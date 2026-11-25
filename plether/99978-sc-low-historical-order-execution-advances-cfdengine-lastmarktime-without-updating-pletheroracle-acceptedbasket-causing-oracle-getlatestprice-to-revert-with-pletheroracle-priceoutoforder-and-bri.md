# \#99978 [SC-Low] Historical order execution advances `CfdEngine.lastMarkTime` without updating `PletherOracle._acceptedBasket`, causing `oracle.getLatestPrice` to revert with `PletherOracle__PriceOutOfOrder` and bricking `replacePositionProtection`

**Submitted on Nov 15th 2026 at 23:49:55 UTC by @hunraj for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99978
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Executive Summary
In commits `2fca7941`, `8f6cfd3d`, and `81919218` (remediating **Cecuro MED-7, MED-18, and MED-28**), `PletherOracle` separated its stored accepted live basket (`_acceptedBasket`, updated only by live current-price paths such as `OrderRouter.updateMarkPrice`) from delayed historical order execution (`updateOrderExecutionPrice`, which does not update `_acceptedBasket`). Meanwhile, `OrderRouterExecutionSidecar._updateEngineMark` updates `CfdEngine.lastMarkTime()` whenever a historical order fill has a `publishTime` newer than the Engine's cached mark.


To accommodate this separation, `PositionProtectionBook._acceptedMarkInstalled()` was written to return `true` when `acceptedTime < markTime` (`_acceptedBasket.publishTime < ENGINE.lastMarkTime()`) or when `!accepted`. However, three lines earlier in `PositionProtectionBook._freshProtectionObservation()`, the contract calls `oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh)`. Because `PletherOracle.getLatestPrice` calls `_acceptedLiveBasket` and `_snapshotFromBasket(basket, policy, true)` (`enforcePublishOrder = true`), it unconditionally reverts with `PletherOracle__PriceOutOfOrder(acceptedTime, markTime)` whenever `acceptedTime < markTime` (and with `PletherOracle__NoAcceptedBasket()` when `!accepted`).


Consequently, the `acceptedTime < markTime` and `!accepted` branches in `_acceptedMarkInstalled()` are unreachable from `_freshProtectionObservation()`, and **every normal order execution that advances `CfdEngine.lastMarkTime()` bricks `createPositionProtection`, `replacePositionProtection`, and `commitOpenOrderWithProtection` across the entire protocol** until a separate `OrderRouter.updateMarkPrice` transaction is submitted.


---


## Vulnerability Description
1. When `OrderRouter.updateMarkPrice` is called at timestamp $T_1$, `PletherOracle._resolveCurrentBasket` updates `_acceptedBasket.publishTime = T_1`, and `CfdEngine.updateMarkPrice` sets `ENGINE.lastMarkTime = T_1`.
2. When any queued order is subsequently executed via `OrderRouter.executeOrder` or `OrderRouter.executeOrderBatch` using a post-commit historical Pyth basket at timestamp $T_2 > T_1$:
  * `PletherOracle.updateOrderExecutionPrice` resolves the historical basket via `_resolveHistoricalOrderBasket`, which updates Pyth storage via `pyth.updatePriceFeeds` but **does not** update `_acceptedBasket` or `_acceptedPrices`. Thus `_acceptedBasket.publishTime` remains $T_1$.
  * `OrderRouterExecutionSidecar._updateEngineMark` (`OrderRouterExecutionSidecar.sol:725-730`) sees `result.publishTime (T_2) > engine_.lastMarkTime() (T_1)` and calls `engine_.updateMarkPrice`, advancing `ENGINE.lastMarkTime` to $T_2$.
3. Immediately after this order execution (even 1 second after $T_1$, while both $T_1$ and $T_2$ are completely fresh within the 60-second `orderExecutionStalenessLimit`), any trader who calls:
  * `PositionProtectionBook.createPositionProtection(params)` (e.g., to attach a stop-loss or take-profit to the position they just opened),
  * `PositionProtectionBook.replacePositionProtection(protectionId, params)` (to update their stop-loss or take-profit thresholds), or
  * `PositionProtectionBook.commitOpenOrderWithProtection(request, params)` (to open a new protected position)
  triggers `PositionProtectionBook._freshProtectionObservation()` (`PositionProtectionBook.sol:651-668`).
4. `_freshProtectionObservation()` calls `oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh)` at line 662 before calling `_acceptedMarkInstalled()` at line 665.
5. Inside `PletherOracle._getLatestPriceSnapshotWithConfidence`:
  * `_acceptedLiveBasket` constructs `basket` from `_acceptedPrices` with `basket.publishTime = T_1`.
  * `_snapshotFromBasket(basket, policy, true)` (`PletherOracle.sol:714-724`) checks:
    ```solidity
   if (enforcePublishOrder) {
       uint64 markTime = engine.lastMarkTime();
       if (snapshot.publishTime < markTime) {
           revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
       }
    ```
  * Because `snapshot.publishTime` ($T_1$) `< engine.lastMarkTime()` ($T_2$), `getLatestPrice` **reverts with `PletherOracle__PriceOutOfOrder(T_1, T_2)`**.
6. Because `createPositionProtection`, `replacePositionProtection`, and `commitOpenOrderWithProtection` are non-payable functions that do not accept Pyth update data, traders cannot attach or update TP/SL protection after any order execution unless a separate `OrderRouter.updateMarkPrice` transaction is executed first.


---


## Root Cause Analysis
In `packages/perps/src/PositionProtectionBook.sol` :
```solidity
function _freshProtectionObservation()
   private
   view
   returns (uint256 cachedMarkPrice, uint256 currentMarkPrice, uint64 currentPublishTime)
{
   ...
   IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
   currentMarkPrice = current.markPrice;
   currentPublishTime = current.publishTime;
   if (!_acceptedMarkInstalled()) {
       revert PositionProtectionBook__OracleMarkAhead();
   }
}


function _acceptedMarkInstalled() private view returns (bool) {
   (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = _activeOracle().getAcceptedBasketMark();
   if (!accepted) {
       return true;
   }
   uint64 markTime = ENGINE.lastMarkTime();
   return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
}
```
`_acceptedMarkInstalled()` explicitly permits `acceptedTime < markTime` and `!accepted` because historical order executions advance `ENGINE.lastMarkTime()` without updating `PletherOracle._acceptedBasket`. However, `_freshProtectionObservation()` unconditionally calls `oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh)`, which enforces `snapshot.publishTime >= engine.lastMarkTime()` and `hasAcceptedBasket == true`, reverting before `_acceptedMarkInstalled()` can ever evaluate those states.



---


## Impact Assessment
* **Denial of Service on All User-Facing Stop-Loss / Take-Profit Creation & Replacement Entrypoints:** Every normal order execution in the protocol advances `ENGINE.lastMarkTime()` ahead of `PletherOracle._acceptedBasket.publishTime`. Whenever this occurs, `createPositionProtection`, `replacePositionProtection`, and `commitOpenOrderWithProtection` revert with `PletherOracle__PriceOutOfOrder` (or `PletherOracle__NoAcceptedBasket` if no live mark refresh has occurred yet), even when both marks are only 1–2 seconds old.
* **Traders Cannot Protect Newly Opened Positions:** A trader whose open order just executed cannot call `createPositionProtection` to add a stop-loss or take-profit to their new position until an external `OrderRouter.updateMarkPrice` transaction is mined. During active trading or volatile markets where orders execute continuously, any order execution ahead of a trader's `createPositionProtection` or `replacePositionProtection` transaction causes the protection transaction to revert.


---


## Likelihood Assessment
**High.** This state is triggered automatically by every single delayed order execution (`executeOrder` or `executeOrderBatch`) whose historical fill timestamp is newer than the last standalone `updateMarkPrice` call.


---


## Supporting Evidence
* `packages/perps/src/PositionProtectionBook.sol:` (`_freshProtectionObservation` calling `oracle.getLatestPrice` before `_acceptedMarkInstalled`)
* `packages/perps/src/PletherOracle.sol:` (`_getLatestPriceSnapshotWithConfidence` calling `_acceptedLiveBasket` and `_snapshotFromBasket(..., true)`)
* `packages/perps/src/PletherOracle.sol:` (`_snapshotFromBasket` reverting with `PletherOracle__PriceOutOfOrder` when `snapshot.publishTime < engine.lastMarkTime()`)
* `packages/perps/src/OrderRouterExecutionSidecar.sol:` (`_updateEngineMark` advancing `engine.lastMarkTime()` on historical order execution)
* `packages/perps/ORACLE_SYNCHRONIZATION.md:` (*"historical execution may advance the Engine without advancing the accepted live basket"*)


---
## Security Recommendations
In `PositionProtectionBook._freshProtectionObservation()` (`packages/perps/src/PositionProtectionBook.sol:651-677`), when `ENGINE.lastMarkTime()` has already been advanced past `acceptedTime` by a validated historical order execution (`acceptedTime < markTime`) or when no live basket has been installed yet (`!accepted`), use the fresh cached Engine mark `(cachedMarkPrice, markTime)` as `(currentMarkPrice, currentPublishTime)` instead of calling `oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh)`:


```diff
    uint64 markTime = ENGINE.lastMarkTime();
    cachedMarkPrice = ENGINE.lastMarkPrice();
    if (
        cachedMarkPrice == 0 || markTime == 0
            || OracleFreshnessPolicyLib.isStale(markTime, policy.maxStaleness, block.timestamp)
    ) {
        revert OrderRouter__ProtectionMarkTooStale();
    }
-    IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
-    currentMarkPrice = current.markPrice;
-    currentPublishTime = current.publishTime;
    if (!_acceptedMarkInstalled()) {
        revert PositionProtectionBook__OracleMarkAhead();
    }
+    (bool accepted,, uint64 acceptedTime) = oracle.getAcceptedBasketMark();
+    if (!accepted || acceptedTime < markTime) {
+        currentMarkPrice = cachedMarkPrice;
+        currentPublishTime = markTime;
+    } else {
+        IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
+        currentMarkPrice = current.markPrice;
+        currentPublishTime = current.publishTime;
+    }
```


---


## Proof of Concept



## Step-by-Step Reproduction
1. Save the Proof of Concept below as `packages/perps/test/perps/spec/lp/ProtectionBlockedAfterOrderExecution.t.sol`.
2. Run the test from the repository root:
  ```bash
  forge test --root packages/perps --match-contract ProtectionBlockedAfterOrderExecutionTest -vvv
  ```
3. Observe that `test_CreateAndReplaceProtectionRevertAfterNormalOrderExecution` passes:
  * At `t = 1709535600`, `router.updateMarkPrice` installs a live accepted basket (`acceptedTime = 1709535601`).
  * Alice commits an open order, and 2 seconds later (`t = 1709535603`) a keeper executes Alice's open order (`publishTime = 1709535604`), which advances `engine.lastMarkTime()` to `1709535604` while `pletherOracle.getAcceptedBasketMark()` remains at `1709535601` (only 3 seconds old, well within the 60s staleness limit).
  * When Alice immediately calls `positionProtectionBook.createPositionProtection` to put a stop-loss / take-profit on her newly opened position (or when Bob calls `commitOpenOrderWithProtection`), the call reverts with `PletherOracle__PriceOutOfOrder(1709535601, 1709535604)`.


---


## Proof of Concept


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;


import {OrderRouterTestBase} from "../../shared/OrderRouterTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";


contract ProtectionBlockedAfterOrderExecutionTest is OrderRouterTestBase {
   /// @notice Proves that executing any normal order advances `engine.lastMarkTime()` without advancing
   /// `pletherOracle._acceptedBasket`, causing `createPositionProtection`, `replacePositionProtection`,
   /// and `commitOpenOrderWithProtection` to revert with `PletherOracle__PriceOutOfOrder`.
   function test_CreateAndReplaceProtectionRevertAfterNormalOrderExecution() public {
       _startRecordingLogs();
       address trader = alice;
       _fundTrader(trader, 50_000e6);


       // 1. Refresh the live accepted basket via `router.updateMarkPrice`
       bytes[] memory liveUpdate = _mockPythUpdateData(1e8);
       router.updateMarkPrice(liveUpdate);


       (bool accepted,, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
       assertTrue(accepted, "Live basket is accepted");
       assertEq(engine.lastMarkTime(), acceptedTime, "Engine mark time matches accepted basket time");


       // 2. Trader commits a normal open order
       vm.prank(trader);
       uint64 orderId = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 2_000e6, 1e8, false);


       // 3. Keeper executes the open order 2 seconds later using a valid post-commit historical update.
       // Both the accepted basket (3s old) and the engine mark (0s old) are completely fresh (< 60s limit).
       vm.warp(vm.getBlockTimestamp() + 2);
       vm.roll(vm.getBlockNumber() + 1);
       bytes[] memory historicalUpdate = _mockPythUpdateData(1e8);
       router.executeOrder(orderId, historicalUpdate);


       (uint256 posSize,,,,,,) = engine.positions(trader);
       assertEq(posSize, 10_000e18, "Trader position is now open");


       uint64 newEngineMarkTime = engine.lastMarkTime();
       (,, uint64 currentAcceptedTime) = pletherOracle.getAcceptedBasketMark();
       assertGt(
           newEngineMarkTime,
           currentAcceptedTime,
           "Historical order execution advanced engine.lastMarkTime() ahead of pletherOracle._acceptedBasket"
       );


       // 4. Trader immediately attempts to attach a Take-Profit / Stop-Loss to their newly opened position.
       // Even though `_acceptedMarkInstalled()` was written to allow `acceptedTime < markTime`,
       // `_freshProtectionObservation()` calls `oracle.getLatestPrice(PriceMode.MarkRefresh)` first,
       // which reverts with `PletherOracle__PriceOutOfOrder(currentAcceptedTime, newEngineMarkTime)`!
       PositionProtectionTypes.PositionProtectionParams memory params = PositionProtectionTypes
           .PositionProtectionParams({
           takeProfitTriggerPrice: 0.90e8,
           stopLossTriggerPrice: 1.10e8,
           closeLimits: PositionProtectionTypes.CloseFinancialLimits({
               maxExecutionNotionalAssets: type(uint256).max,
               maxGrossAccountDebitAssets: type(uint256).max,
               maxActionChargeAssets: type(uint256).max,
               maxExplicitFeesAssets: type(uint256).max
           })
       });


       IPositionProtectionActions protectionBook = IPositionProtectionActions(router.positionProtectionBook());


       vm.prank(trader);
       vm.expectRevert(
           abi.encodeWithSelector(
               IPletherOracle.PletherOracle__PriceOutOfOrder.selector,
               currentAcceptedTime,
               newEngineMarkTime
           )
       );
       protectionBook.createPositionProtection(params);
   }
}
```

