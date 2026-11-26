# \#99598 [SC-Insight] Liquidation previews understate the account's post-liquidation settlement by the keeper share when the account liquidates itself

**Submitted on Nov 14th 2026 at 12:31:26 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99598
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`CfdEngineLens.previewLiquidation` and the Router-aware `previewRouterLiquidation` return `settlementRetainedAssets`, which the interface documents as the "total projected account settlement after liquidation debits and any immediate payout". Liquidation is permissionless, and the Router credits the keeper share to `msg.sender`, so an account can liquidate itself. The previews always subtract the full liquidation charge and take no executor input. For a self-liquidation, the projected account total is therefore wrong by exactly the keeper share: 5 bp of the liquidated notional at release parameters. In the PoC, a 100,000 USDG LONG liquidated after a 12 bp basket move is projected to keep 90.11 USDG and actually keeps 140.11 USDG.

## Vulnerability Details

The planner computes the retained balance by deducting the whole charge, including the keeper share ([`CfdEnginePlanLib.sol` L1546-L1550](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1546-L1550)):

```solidity
uint256 settlementAfterDebitsAssets = snap.accountBuckets.settlementBalanceAssets
    - delta.pricePnlPledgeConsumedAssets - delta.actionChargeCollectedAssets - delta.liquidationChargeAssets; // <-@ keeper share deducted
delta.settlementRetainedAssets =
    settlementAfterDebitsAssets + (delta.freshPayoutIsImmediate ? delta.freshTraderPayoutAssets : 0);
```

`CfdEngineLens._liquidationPreviewFromSnapshot` copies this value into `LiquidationPreview.settlementRetainedAssets` ([`CfdEngineLens.sol` L636](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineLens.sol#L620-L640)). Neither `previewLiquidation(account, oraclePrice)` nor `previewRouterLiquidation(account, oraclePrice)` has a keeper parameter ([L201-L222](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineLens.sol#L201-L222)).

Execution depends on the caller:

- `executeLiquidation` and `executeLiquidationBatch` pass `msg.sender` as the keeper ([`OrderLiquidationBatchLogic.sol` L728, L783](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L706-L809)).
- The Clearinghouse debits the account for the full charge, then runs `settlementBalances[execution.keeper] += execution.keeperBountyAssets` ([`MarginClearinghouse.sol` L1252, L1268](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1234-L1269)).
- When keeper == account, the account therefore ends at `settlementRetainedAssets + keeperBountyAssets`.
- The Engine already treats this alias as a supported path: it skips the keeper checkpoint and curve sync when `keeper != delta.account` is false ([`CfdEngine.sol` L1331, L1336](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1324-L1347)).

This breaks properties the protocol states:

- [`ICfdEngineTypes.sol` L397-L399](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/ICfdEngineTypes.sol#L393-L399): "Total projected account settlement after liquidation debits and any immediate payout".
- [`ACCOUNTING_SPEC.md` L1135-L1140](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L1126-L1140): "Retained settlement is an account-balance projection".
- [`SECURITY.md` L356](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L356), under Critical Protocol Invariants: "Preview/live parity | Close and liquidation preview math should match live execution semantics".
- The repository's conservation invariant `_assertInvariant_TerminalEventsMatchResidualAndPriceWriteoffAccounting` ([`PerpEconomicConservationInvariant.t.sol` L429-L446](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/invariant/properties/PerpEconomicConservationInvariant.t.sol#L429-L446)) asserts that final settlement plus trader claim equals `preview.settlementRetainedAssets + preview.traderClaimBalanceAssets` ([`PerpAccountingHandler.sol` L577-L579](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/invariant/handlers/PerpAccountingHandler.sol#L563-L594)). Its handler always liquidates from the handler contract (L583), so this path is never exercised. The PoC evaluates the same expression: it holds for a distinct keeper and fails by exactly the keeper share for a self-liquidation.

The protocol models this exact alias for order execution. The policy evaluator sets `bountyReturnsToAccount = context.executor == order.account` and does not deduct the bounty from the projected post-settlement balance ([`CfdOrderPolicyEvaluatorBase.sol` L110-L121, L150](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L110-L150)), and the repository tests it ([`CfdOrderPolicyEvaluatorParity.t.sol` L82-L95](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/differential/CfdOrderPolicyEvaluatorParity.t.sol#L82-L95)). The liquidation previews have no executor input, so no caller can obtain the documented total from them. Their documented caveats ([`ICfdEngineLens.sol` L159-L163](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/ICfdEngineLens.sol#L159-L167): oracle policy, intervening transactions, other batch accounts, callbacks, gas) do not include the caller's identity.

## Impact Details

**Low: contract fails to deliver promised returns, but doesn't lose value.**

- The in-scope liquidation previews return a wrong account total whenever the account is its own liquidator. This includes the Router-aware preview, which the specification tells integrators to use.
- The error equals the keeper share: 50% of the liquidation charge, or 5 bp of notional.
- An account or front end that uses the documented projection sees the same retained balance whether it liquidates itself or waits for a third-party keeper.
- The repository's own residual-conservation property fails on this path.

| Path (same state and mark) | Preview `settlementRetainedAssets` | Distinct keeper: account after | Account liquidates itself: account after |
|---|---:|---:|---:|
| `executeLiquidation`, 400 USDG margin, 12 bp adverse move | 90.112786 | 90.112786 | **140.112786** |
| `executeLiquidation`, margin exhausted | 0 | 0 | **50.000000** |
| `executeLiquidationBatch`, 1,000 USDG free settlement (`previewRouterLiquidation` also 1,000) | 1,000.000000 | 1,000.000000 | **1,050.000000** |

**Recommendation:** pass the executor to the liquidation previews, or return both projections. Add `keeperBountyAssets` to `settlementRetainedAssets` when the executor is the account, as the order evaluator already does with `bountyReturnsToAccount`.

## References

- [CfdEnginePlanLib.sol L1546-L1550](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1546-L1550)
- [CfdEngineLens.sol L201-L222](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineLens.sol#L201-L222) and [L620-L640](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineLens.sol#L620-L640)
- [ICfdEngineTypes.sol L393-L399](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/ICfdEngineTypes.sol#L393-L399)
- [ICfdEngineLens.sol L159-L167](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/ICfdEngineLens.sol#L159-L167)
- [OrderLiquidationBatchLogic.sol L706-L809](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L706-L809)
- [MarginClearinghouse.sol L1234-L1269](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1234-L1269)
- [CfdEngine.sol L1324-L1347](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1324-L1347)
- [CfdOrderPolicyEvaluatorBase.sol L110-L150](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L110-L150)
- [ACCOUNTING_SPEC.md L1126-L1140](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L1126-L1140)
- [SECURITY.md L356](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L356)
- [PerpEconomicConservationInvariant.t.sol L429-L446](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/invariant/properties/PerpEconomicConservationInvariant.t.sol#L429-L446)
- [PerpAccountingHandler.sol L563-L594](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/invariant/handlers/PerpAccountingHandler.sol#L563-L594)
- [CfdOrderPolicyEvaluatorParity.t.sol L82-L95](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/differential/CfdOrderPolicyEvaluatorParity.t.sol#L82-L95)

## Proof of Concept

# Proof of Concept

Place the test below at `packages/perps/test/poc/SelfLiquidationPreviewAlias.t.sol` and run, from the repository root:

```sh
forge test --root packages/perps --match-path test/poc/SelfLiquidationPreviewAlias.t.sol -vv
```

The test runs against the unmodified contracts at the audited commit and uses the repository's own `BasePerpTest`; only the Pyth price feed and USDG are mocked.

## Test

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineLens} from "@plether/perps/interfaces/ICfdEngineLens.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Demonstrates that liquidation preview subtracts the keeper allocation even when the account self-liquidates.
/// @dev Both branches use the public Router liquidation entrypoint and the release risk parameters.
contract SelfLiquidationPreviewAliasTest is BasePerpTest {

    address internal constant TARGET = address(0xA11CE);
    address internal constant CONTROL_KEEPER = address(0xB0B);

    uint256 internal constant ENTRY_PRICE = 100_000_000;

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
        _configureReleaseSixFeeds();
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen spread");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior ratio");
    }

    function test_PublicSelfLiquidation_CreditsKeeperShareAbovePreviewRetainedBalance() public {
        _fundTrader(TARGET, 10_000e6);
        uint64 t = uint64(block.timestamp);
        vm.prank(CONTROL_KEEPER);
        router.updateMarkPrice(_sixFeedData(10_000, t, t - 1));
        uint64 id = _commitOpen();
        vm.warp(t + 1);
        vm.roll(block.number + 1);
        vm.prank(CONTROL_KEEPER);
        OrderV3Types.ExecutionResult memory opened = router.executeOrder(id, _sixFeedData(10_000, t + 1, t));
        assertEq(uint8(opened.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public V3 open");
        _withdrawAllFree(TARGET);

        vm.warp(t + 2);
        vm.roll(block.number + 1);
        bytes[] memory priceData = _sixFeedData(10_900, t + 2, t + 1);
        vm.prank(CONTROL_KEEPER);
        router.updateMarkPrice(priceData);
        uint256 liquidationPrice = engine.lastMarkPrice();
        ICfdEngineTypes.LiquidationPreview memory preview =
            engineLens.previewLiquidation(TARGET, liquidationPrice);
        assertTrue(preview.liquidatable, "fixture must be liquidatable");
        assertGt(preview.keeperBountyAssets, 0, "fixture needs a nonzero keeper allocation");

        uint256 custodyBefore = _systemCustody();
        uint256 controlKeeperBalanceBefore = clearinghouse.balanceAssets(CONTROL_KEEPER);
        uint256 checkpoint = vm.snapshotState();

        // Control: the keeper is a distinct account, so the target retains exactly the previewed amount.
        vm.prank(CONTROL_KEEPER);
        router.executeLiquidation(TARGET, priceData);

        uint256 controlTargetBalance = clearinghouse.balanceAssets(TARGET);
        uint256 controlKeeperBalance = clearinghouse.balanceAssets(CONTROL_KEEPER);
        uint256 controlTreasuryBalance = clearinghouse.balanceAssets(engine.protocolTreasury());
        uint256 controlPoolAssets = pool.totalAssets();

        assertEq(controlTargetBalance, preview.settlementRetainedAssets, "control must match retained preview");
        assertEq(
            controlKeeperBalance - controlKeeperBalanceBefore,
            preview.keeperBountyAssets,
            "control keeper must receive the previewed allocation"
        );
        assertEq(_systemCustody(), custodyBefore, "control must conserve USDG custody");
        assertEq(_positionSize(TARGET), 0, "control liquidation must complete");

        assertTrue(vm.revertToState(checkpoint), "fixture reset failed");

        // Treatment: the target is also the keeper. Application debits the full charge, then credits its keeper share
        // back to the same settlement balance even though settlementRetainedAssets subtracted that share.
        vm.prank(TARGET);
        router.executeLiquidation(TARGET, priceData);

        uint256 treatmentTargetBalance = clearinghouse.balanceAssets(TARGET);
        uint256 treatmentTreasuryBalance = clearinghouse.balanceAssets(engine.protocolTreasury());
        uint256 treatmentPoolAssets = pool.totalAssets();

        assertEq(
            treatmentTargetBalance,
            preview.settlementRetainedAssets + preview.keeperBountyAssets,
            "self-liquidation retains the keeper share above preview"
        );
        assertEq(treatmentPoolAssets, controlPoolAssets, "LP allocation must be keeper-identity independent");
        assertEq(
            treatmentTreasuryBalance,
            controlTreasuryBalance,
            "protocol allocation must be keeper-identity independent"
        );
        assertEq(
            treatmentTargetBalance + treatmentTreasuryBalance + treatmentPoolAssets,
            controlTargetBalance + controlKeeperBalance + controlTreasuryBalance + controlPoolAssets
                - controlKeeperBalanceBefore,
            "the same total liquidation value must be allocated in both branches"
        );
        assertEq(_systemCustody(), custodyBefore, "self-liquidation must conserve USDG custody");
        assertEq(_positionSize(TARGET), 0, "self-liquidation must complete");
        console2.log("preview retained", preview.settlementRetainedAssets);
        console2.log("keeper share", preview.keeperBountyAssets);
        console2.log("distinct-keeper actual retained", controlTargetBalance);
        console2.log("self-keeper actual retained", treatmentTargetBalance);
        console2.log("liquidation basket price", liquidationPrice);
    }

    /// @notice Same alias through the batch entrypoint, with free settlement left in the account so the preview
    ///         projects a nonzero retained balance. The batch credits the keeper share to `msg.sender` for every item.
    function test_BatchSelfLiquidation_WithFreeSettlement_RetainsKeeperShareAbovePreview() public {
        _fundTrader(TARGET, 10_000e6);
        uint64 t = uint64(block.timestamp);
        vm.prank(CONTROL_KEEPER);
        router.updateMarkPrice(_sixFeedData(10_000, t, t - 1));
        uint64 id = _commitOpen();
        vm.warp(t + 1);
        vm.roll(block.number + 1);
        vm.prank(CONTROL_KEEPER);
        OrderV3Types.ExecutionResult memory opened = router.executeOrder(id, _sixFeedData(10_000, t + 1, t));
        assertEq(uint8(opened.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public V3 open");
        // Leave exactly 1,000 USDG of free settlement in the account.
        uint256 free = _freeSettlementAssets(TARGET);
        vm.prank(TARGET);
        clearinghouse.withdraw(TARGET, free - 1_000e6);
        assertEq(_freeSettlementAssets(TARGET), 1_000e6, "1,000 USDG left free");

        vm.warp(t + 2);
        vm.roll(block.number + 1);
        bytes[] memory priceData = _sixFeedData(10_900, t + 2, t + 1);
        vm.prank(CONTROL_KEEPER);
        router.updateMarkPrice(priceData);
        uint256 liquidationPrice = engine.lastMarkPrice();
        ICfdEngineTypes.LiquidationPreview memory preview =
            engineLens.previewLiquidation(TARGET, liquidationPrice);
        assertTrue(preview.liquidatable, "fixture must be liquidatable");
        assertGt(preview.settlementRetainedAssets, 0, "preview projects a nonzero retained balance");
        assertGt(preview.keeperBountyAssets, 0, "nonzero keeper allocation");
        // The Router-aware preview recommended by the specification projects the same retained balance.
        ICfdEngineLens.RouterLiquidationPreview memory routerPreview =
            engineLens.previewRouterLiquidation(TARGET, liquidationPrice);
        assertEq(
            routerPreview.liquidation.settlementRetainedAssets,
            preview.settlementRetainedAssets,
            "Router-aware preview projects the same retained balance"
        );

        address[] memory accounts = new address[](1);
        accounts[0] = TARGET;
        uint256 checkpoint = vm.snapshotState();

        vm.prank(CONTROL_KEEPER);
        router.executeLiquidationBatch(accounts, priceData);
        uint256 controlTargetBalance = clearinghouse.balanceAssets(TARGET);
        uint256 controlPoolAssets = pool.totalAssets();
        assertEq(_positionSize(TARGET), 0, "control batch liquidation must complete");
        assertEq(controlTargetBalance, preview.settlementRetainedAssets, "control matches the preview");

        assertTrue(vm.revertToState(checkpoint), "fixture reset failed");

        vm.prank(TARGET);
        router.executeLiquidationBatch(accounts, priceData);
        uint256 treatmentTargetBalance = clearinghouse.balanceAssets(TARGET);
        assertEq(_positionSize(TARGET), 0, "self batch liquidation must complete");
        assertEq(
            treatmentTargetBalance,
            preview.settlementRetainedAssets + preview.keeperBountyAssets,
            "self-keeper retains the keeper share above the preview"
        );
        assertEq(pool.totalAssets(), controlPoolAssets, "LP allocation unchanged by keeper identity");
        console2.log("batch preview retained", preview.settlementRetainedAssets);
        console2.log("batch router-aware preview retained", routerPreview.liquidation.settlementRetainedAssets);
        console2.log("batch keeper share", preview.keeperBountyAssets);
        console2.log("batch distinct-keeper actual retained", controlTargetBalance);
        console2.log("batch self-keeper actual retained", treatmentTargetBalance);
    }

    /// @notice Same comparison for a thin-margin position liquidated one basis point past its liquidation boundary,
    ///         checked with the expectation the repository's conservation-invariant handler builds from the preview:
    ///         final settlement plus trader claim == previewed retained settlement plus previewed trader claim.
    function test_BoundaryMoveSelfLiquidation_BreaksRepositoryResidualEquality() public {
        _fundTrader(TARGET, 10_000e6);
        uint64 t = uint64(block.timestamp);
        vm.prank(CONTROL_KEEPER);
        router.updateMarkPrice(_sixFeedData(10_000, t, t - 1));
        uint64 id = _commitOpenWithMargin(400e6);
        vm.warp(t + 1);
        vm.roll(block.number + 1);
        vm.prank(CONTROL_KEEPER);
        OrderV3Types.ExecutionResult memory opened = router.executeOrder(id, _sixFeedData(10_000, t + 1, t));
        assertEq(uint8(opened.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public V3 open");
        _withdrawAllFree(TARGET);

        vm.warp(t + 2);
        vm.roll(block.number + 1);
        // Smallest basket level, in basis points of the entry level, at which the lens reports the account liquidatable.
        uint256 lo = 10_000;
        uint256 hi = 11_000;
        assertFalse(engineLens.previewLiquidation(TARGET, lo * 1e4).liquidatable, "healthy at the entry level");
        assertTrue(engineLens.previewLiquidation(TARGET, hi * 1e4).liquidatable, "liquidatable after a 10% move");
        while (hi - lo > 1) {
            uint256 mid = (lo + hi) / 2;
            if (engineLens.previewLiquidation(TARGET, mid * 1e4).liquidatable) {
                hi = mid;
            } else {
                lo = mid;
            }
        }
        uint256 levelBps = hi + 1;
        bytes[] memory priceData = _sixFeedData(levelBps, t + 2, t + 1);
        vm.prank(CONTROL_KEEPER);
        router.updateMarkPrice(priceData);
        uint256 liquidationPrice = engine.lastMarkPrice();
        ICfdEngineTypes.LiquidationPreview memory preview = engineLens.previewLiquidation(TARGET, liquidationPrice);
        assertTrue(preview.liquidatable, "liquidatable at the published boundary mark");
        assertGt(preview.settlementRetainedAssets, 0, "preview projects a nonzero retained balance");
        assertGt(preview.keeperBountyAssets, 0, "nonzero keeper allocation");

        // Expected terminal residual exactly as the repository's conservation-invariant handler computes it.
        uint256 expectedFinalResidual = preview.settlementRetainedAssets + preview.traderClaimBalanceAssets;
        uint256 checkpoint = vm.snapshotState();

        vm.prank(CONTROL_KEEPER);
        router.executeLiquidation(TARGET, priceData);
        assertEq(_positionSize(TARGET), 0, "control liquidation must complete");
        uint256 controlResidual = clearinghouse.balanceAssets(TARGET) + engine.traderClaimBalanceAssets(TARGET);
        assertEq(controlResidual, expectedFinalResidual, "distinct keeper: the repository residual equality holds");

        assertTrue(vm.revertToState(checkpoint), "fixture reset failed");

        vm.prank(TARGET);
        router.executeLiquidation(TARGET, priceData);
        assertEq(_positionSize(TARGET), 0, "self-liquidation must complete");
        uint256 selfResidual = clearinghouse.balanceAssets(TARGET) + engine.traderClaimBalanceAssets(TARGET);
        assertTrue(selfResidual != expectedFinalResidual, "self-keeper: the repository residual equality fails");
        assertEq(
            selfResidual,
            expectedFinalResidual + preview.keeperBountyAssets,
            "self-keeper residual exceeds the projection by exactly the keeper share"
        );
        console2.log("boundary adverse basket move (bp)", levelBps - 10_000);
        console2.log("boundary liquidation basket price", liquidationPrice);
        console2.log("boundary preview retained", preview.settlementRetainedAssets);
        console2.log("boundary keeper share", preview.keeperBountyAssets);
        console2.log("boundary distinct-keeper final residual", controlResidual);
        console2.log("boundary self-keeper final residual", selfResidual);
    }

    function _commitOpen() private returns (uint64 id) {
        id = _commitOpenWithMargin(2000e6);
    }

    function _commitOpenWithMargin(
        uint256 marginAssets
    ) private returns (uint64 id) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: CfdTypes.Side.LONG,
            sizeDelta: 100_000e18,
            marginDelta: marginAssets,
            targetPrice: 1,
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + uint64(window),
                executionWindowSeconds: window,
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
        vm.prank(TARGET);
        id = router.commitOrder(request);
    }

    function _configureReleaseSixFeeds() private {
        PletherOracle six = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), _feedIds(), _weights(), _basePrices(), _inversions()
        );
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig({pletherOracle: address(six)}));
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.basketMaxConfidenceRatioBps = 10;
        rc.adverseConfidenceMultiplierBps = 2500;
        rc.minOpenNotionalAssets = 1_000e6;
        routerAdmin.proposeRouterConfig(rc);
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = 40_000_000e6;
        pc.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(pc);
        vm.warp(block.timestamp + 72 hours + 1);
        routerAdmin.finalizeOracleConfig();
        routerAdmin.finalizeRouterConfig();
        pool.finalizePoolConfig();
        pletherOracle = six;
    }

    function _sixFeedData(uint256 priceBps, uint64 publishTime, uint64 prevPublishTime)
        private pure returns (bytes[] memory data)
    {
        bytes32[] memory ids = _feedIds();
        bool[] memory inverted = _inversions();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            uint256 baseRaw = _rawPrice(i);
            uint256 raw = inverted[i] ? baseRaw * 10_000 / priceBps : baseRaw * priceBps / 10_000;
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(raw)), 0, -8, publishTime, prevPublishTime)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _feedIds() private pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _weights() private pure returns (uint256[] memory w) {
        w = new uint256[](6);
        w[0] = 576e15; w[1] = 136e15; w[2] = 119e15;
        w[3] = 91e15; w[4] = 42e15; w[5] = 36e15;
    }

    function _basePrices() private pure returns (uint256[] memory x) {
        x = new uint256[](6);
        x[0] = 117_500_000; x[1] = 638_000; x[2] = 134_480_000;
        x[3] = 72_880_000; x[4] = 10_860_000; x[5] = 126_100_000;
    }

    function _inversions() private pure returns (bool[] memory inv) {
        inv = new bool[](6);
        inv[1] = true; inv[3] = true; inv[4] = true; inv[5] = true;
    }

    function _rawPrice(uint256 i) private pure returns (uint256) {
        if (i == 0) return 117_500_000;
        if (i == 1) return uint256(1e16) / 638_000;
        if (i == 2) return 134_480_000;
        if (i == 3) return uint256(1e16) / 72_880_000;
        if (i == 4) return uint256(1e16) / 10_860_000;
        return uint256(1e16) / 126_100_000;
    }

    function _withdrawAllFree(
        address account
    ) private {
        uint256 freeAssets = clearinghouse.getFreeBuyingPowerAssets(account);
        if (freeAssets != 0) {
            vm.prank(account);
            clearinghouse.withdraw(account, freeAssets);
        }
    }

    function _systemCustody() private view returns (uint256) {
        return settlementToken.balanceOf(address(pool)) + settlementToken.balanceOf(address(clearinghouse));
    }

    function _positionSize(
        address account
    ) private view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

}
```

## Output

```text
Ran 3 tests for test/poc/SelfLiquidationPreviewAlias.t.sol:SelfLiquidationPreviewAliasTest
[PASS] test_BatchSelfLiquidation_WithFreeSettlement_RetainsKeeperShareAbovePreview() (gas: 4701418)
Logs:
  batch preview retained 1000000000
  batch router-aware preview retained 1000000000
  batch keeper share 50000000
  batch distinct-keeper actual retained 1000000000
  batch self-keeper actual retained 1050000000

[PASS] test_BoundaryMoveSelfLiquidation_BreaksRepositoryResidualEquality() (gas: 5834900)
Logs:
  boundary adverse basket move (bp) 12
  boundary liquidation basket price 100119987
  boundary preview retained 90112786
  boundary keeper share 50000000
  boundary distinct-keeper final residual 90112786
  boundary self-keeper final residual 140112786

[PASS] test_PublicSelfLiquidation_CreditsKeeperShareAbovePreviewRetainedBalance() (gas: 4603212)
Logs:
  preview retained 0
  keeper share 50000000
  distinct-keeper actual retained 0
  self-keeper actual retained 50000000
  liquidation basket price 109000000

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 18.22ms (21.66ms CPU time)

Ran 1 test suite in 19.42ms (18.22ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```
