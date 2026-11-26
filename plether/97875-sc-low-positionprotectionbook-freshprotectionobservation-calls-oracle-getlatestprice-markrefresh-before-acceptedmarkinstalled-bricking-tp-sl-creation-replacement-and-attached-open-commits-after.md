# \#97875 [SC-Low] PositionProtectionBook._freshProtectionObservation Calls oracle.getLatestPrice(MarkRefresh) Before _acceptedMarkInstalled(), Bricking TP/SL Creation, Replacement, and Attached-Open Commits After Any Historical Order Execution

**Submitted on Nov 3rd 2026 at 17:53:46 UTC by @alpha064 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97875
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

# `PositionProtectionBook._freshProtectionObservation` Calls `oracle.getLatestPrice(MarkRefresh)` Before `_acceptedMarkInstalled()`, Bricking TP/SL Creation, Replacement, and Attached-Open Commits After Any Historical Order Execution

## Summary & Post-Audit Regression Provenance

The final pre-competition security audit (`audits/plether-ai-audit-report-2026-10-26.pdf`) evaluated commit `6d6622f2871dbdfc4904c20da2e1f7662f5763b6` (`fix(perps): use net risk depth and fingerprint calendar changes (#118)`, October 15, 2026). To remediate the audit's oracle and protection findings (`MED-28`), two subsequent post-audit commits modified `packages/perps/src/PositionProtectionBook.sol`:

1. **Commit `819192182833597641504b38cce7751bbcacecb8` (`fix(perps): synchronize historical Pyth and enforce signed live basket (#120)`, October 21, 2026)** introduced `_acceptedMarkInstalled()` (`lines 888-895`) and called it at the end of `_freshCachedProtectionMark()`. Because historical order execution (`OrderRouter.executeOrder` / `executeOrderBatch`) advances `ENGINE.lastMarkTime()` without updating `PletherOracle._acceptedBasket` (`packages/perps/ORACLE_SYNCHRONIZATION.md`, lines 150-151: *"historical execution may advance the Engine without advancing the accepted live basket"*), `_acceptedMarkInstalled()` was explicitly written to return `true` when `!accepted` or `acceptedTime < markTime`:
   ```solidity
   function _acceptedMarkInstalled() private view returns (bool) {
       (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = _activeOracle().getAcceptedBasketMark();
       if (!accepted) {
           return true;
       }
       uint64 markTime = ENGINE.lastMarkTime();
       return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
   }
   ```
2. **Commit `eadc5c587e2f2f1ec26c51132472ab876d6edc15` (`fix(perps): harden trader protection, liquidation pricing, and manual exits (#124)`, sub-commit `fix(perps): validate protection against current oracle publication`, October 23, 2026 — direct ancestor of the in-scope competition commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`)** renamed `_freshCachedProtectionMark()` to `_freshProtectionObservation()` (`lines 862-886`) and inserted an unconditional call to `oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh)` on line 880 **before** `if (!_acceptedMarkInstalled())` on line 883:
   ```diff
   +        IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
   +        currentMarkPrice = current.markPrice;
   +        currentPublishTime = current.publishTime;
            if (!_acceptedMarkInstalled()) {
                revert PositionProtectionBook__OracleMarkAhead();
            }
   ```

Inside `PletherOracle`, `getLatestPrice(PriceMode.MarkRefresh)` calls `_getLatestPriceSnapshotWithConfidence(MarkRefresh)` (`lines 571-580`), which loads `_acceptedLiveBasket` (whose `publishTime` is `acceptedTime = _acceptedBasket.publishTime`) and passes `enforcePublishOrder = true` to `_snapshotFromBasket()` (`lines 878-903`). That function reverts in every state that `_acceptedMarkInstalled()` was designed to allow:
- When `acceptedTime < ENGINE.lastMarkTime()` (after any historical order execution advances `ENGINE.lastMarkTime()`), `_snapshotFromBasket` reverts at line 895 with `PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime)`.
- When `!accepted` (`!hasAcceptedBasket`, e.g., after startup mark initialization via `CfdEngine.initializeInitialMarkPrice` added in commit `88ef395423cf1cfbe5181425b3bcc955f94e8b3a` `#126`, or after historical order execution before the first live mark refresh), `_acceptedLiveBasket` reverts at line 712 with `PletherOracle__NoAcceptedBasket()`.
- When `block.timestamp > acceptedTime + policy.maxStaleness` (`60s`) while `ENGINE.lastMarkTime()` was just refreshed by a historical order execution in the current block, `_acceptedLiveBasket` reverts at line 721 with `PletherOracle__StalePrice`.

Consequently, the `!accepted` and `acceptedTime < markTime` branches of `_acceptedMarkInstalled()` are completely unreachable from `_freshProtectionObservation()`. Whenever any delayed order executes and advances `ENGINE.lastMarkTime()` ahead of `_acceptedBasket.publishTime`, all three trader protection entrypoints—`createPositionProtection`, `replacePositionProtection`, and `commitOpenOrderWithProtection`—revert until a separate live `OrderRouter.updateMarkPrice` transaction is paid for and executed.

---

## Vulnerable Code

### 1. `packages/perps/src/PositionProtectionBook.sol` (`lines 229-231`, `295-296`, `463-464`, and `862-895`)

Link: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L862-L895

```solidity
    function _freshProtectionObservation()
        private
        view
        returns (uint256 cachedMarkPrice, uint256 currentMarkPrice, uint64 currentPublishTime)
    {
        IPletherOracle oracle = _activeOracle();
        IPletherOracle.PolicySnapshot memory policy = oracle.getOrderExecutionPolicy(true);
        if (policy.oracleFrozen) {
            revert OrderRouter__ConditionalTriggerFrozen();
        }
        uint64 markTime = ENGINE.lastMarkTime();
        cachedMarkPrice = ENGINE.lastMarkPrice();
        if (
            cachedMarkPrice == 0 || markTime == 0
                || OracleFreshnessPolicyLib.isStale(markTime, policy.maxStaleness, block.timestamp)
        ) {
            revert OrderRouter__ProtectionMarkTooStale();
        }
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

### 2. `packages/perps/src/PletherOracle.sol` (`lines 565-580`, `706-727`, and `878-903`)

Link: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L565-L580

```solidity
    function _getLatestPriceSnapshot(
        PriceMode mode
    ) internal view returns (PriceSnapshot memory snapshot) {
        (snapshot,) = _getLatestPriceSnapshotWithConfidence(mode);
    }

    function _getLatestPriceSnapshotWithConfidence(
        PriceMode mode
    ) internal view returns (PriceSnapshot memory snapshot, uint256 confidence) {
        PolicySnapshot memory policy = _policyForMode(mode);
        BasketPrice memory basket = policy.oracleFrozen
            ? _frozenBasket(mode, policy.maxStaleness)
            : _acceptedLiveBasket(mode, policy.maxStaleness, _policyPublishTimeDivergence(mode, policy));
        snapshot = _snapshotFromBasket(basket, policy, true);
        confidence = basket.confidence;
    }
```

Link: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L706-L727

```solidity
    function _acceptedLiveBasket(
        PriceMode mode,
        uint256 maxStaleness,
        uint256 maxPublishTimeDivergence
    ) internal view returns (BasketPrice memory basket) {
        if (!hasAcceptedBasket) {
            revert PletherOracle__NoAcceptedBasket();
        }
        PythStructs.PriceFeed[] memory priceFeeds = new PythStructs.PriceFeed[](pythFeedIds.length);
        uint256 len = pythFeedIds.length;

        for (uint256 i = 0; i < len; i++) {
            bytes32 feedId = pythFeedIds[i];
            PythStructs.Price memory p = _acceptedPrices[i];
            if (OracleFreshnessPolicyLib.isStale(p.publishTime, maxStaleness, block.timestamp)) {
                revert PletherOracle__StalePrice(mode, feedId, p.publishTime, maxStaleness, block.timestamp);
            }
            priceFeeds[i] = PythStructs.PriceFeed({id: feedId, price: p, emaPrice: p});
        }

        basket = _computeBasketPriceFromFeeds(mode, priceFeeds, maxPublishTimeDivergence);
    }
```

Link: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L878-L903

```solidity
    function _snapshotFromBasket(
        BasketPrice memory basket,
        PolicySnapshot memory policy,
        bool enforcePublishOrder
    ) internal view returns (PriceSnapshot memory snapshot) {
        snapshot.maxStaleness = policy.maxStaleness;
        snapshot.closeOnly = policy.closeOnly;
        snapshot.oracleFrozen = policy.oracleFrozen;
        snapshot.isFadWindow = policy.isFadWindow;
        snapshot.price = _clampToCap(basket.price);
        snapshot.markPrice = snapshot.price;
        snapshot.publishTime = basket.publishTime;
        snapshot.updateFee = basket.pythFee;

        if (enforcePublishOrder) {
            uint64 markTime = engine.lastMarkTime();
            if (snapshot.publishTime < markTime) {
                revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
            }
            if (snapshot.publishTime == markTime && engine.lastMarkPrice() != snapshot.markPrice) {
                revert PletherOracle__EqualTimeMarkMismatch(
                    snapshot.publishTime, snapshot.markPrice, engine.lastMarkPrice()
                );
            }
        }
    }
```

---

## Root Cause Analysis

1. **Order Execution Advances `ENGINE.lastMarkTime()` Without Updating `PletherOracle._acceptedBasket`**:
   - When `OrderRouter.executeOrder` or `OrderRouter.executeOrderBatch` executes a pending order in live or FAD mode, `PletherOracle._resolveHistoricalOrderBasket()` (`PletherOracle.sol:596-643`) parses the post-commit historical Pyth update and updates Pyth storage, but does **not** write to `_acceptedPrices`, `_acceptedBasket`, or `hasAcceptedBasket`.
   - `OrderRouterExecutionSidecar._updateEngineMark()` (`OrderRouterExecutionSidecar.sol:1053-1061`) then calls `engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime)` whenever `result.publishTime > engine_.lastMarkTime()`.
   - Consequently, after any historical order execution at timestamp $T_{\text{exec}} > T_{\text{accepted}}$, `ENGINE.lastMarkTime()` equals $T_{\text{exec}}$, while `PletherOracle.getAcceptedBasketMark()` still has `acceptedTime = T_{\text{accepted}} < T_{\text{exec}}$ (or `accepted = false` if no live mark refresh has occurred yet).

2. **`_freshProtectionObservation()` Reverts Inside `oracle.getLatestPrice(MarkRefresh)` Before Reaching `_acceptedMarkInstalled()`**:
   - `_freshProtectionObservation()` is called by:
     - `createPositionProtection` -> `_create` (`PositionProtectionBook.sol:229`)
     - `replacePositionProtection` -> `_replace` (`PositionProtectionBook.sol:295`)
     - `commitOpenOrderWithProtection` -> `_stageAttached` (`PositionProtectionBook.sol:463`)
   - In `_freshProtectionObservation()`, lines 872-879 verify that `ENGINE.lastMarkPrice()` (`cachedMarkPrice`) and `ENGINE.lastMarkTime()` (`markTime`) are non-zero and fresh within `policy.maxStaleness`.
   - Line 880 then unconditionally calls `oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh)`.
   - When `!policy.oracleFrozen`, `PletherOracle._getLatestPriceSnapshotWithConfidence` calls `_acceptedLiveBasket()` and `_snapshotFromBasket(basket, policy, true)`.
   - Inside `_snapshotFromBasket(..., true)`, lines 893-895 check:
     ```solidity
     uint64 markTime = engine.lastMarkTime();
     if (snapshot.publishTime < markTime) {
         revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
     }
     ```
   - Because `snapshot.publishTime == acceptedTime < markTime`, `oracle.getLatestPrice(MarkRefresh)` always reverts with `PletherOracle__PriceOutOfOrder(acceptedTime, markTime)` before line 883 (`if (!_acceptedMarkInstalled())`) is ever reached.

---

## Impact

- **Temporary Freezing / Denial of Service of Position Protections (TP/SL)**:
  1. Immediately after a trader's open order is executed via `executeOrder` or `executeOrderBatch`, the trader cannot call `createPositionProtection` to attach Take-Profit or Stop-Loss protection to their newly opened position—even in the exact same block or the very next second while `ENGINE.lastMarkTime()` is completely fresh.
  2. Any trader with an already-armed TP/SL protection (`PositionProtectionStatus.Armed`) or pending-open protection (`PositionProtectionStatus.PendingOpen`) is blocked from updating their Stop-Loss or Take-Profit thresholds via `replacePositionProtection` whenever any other user's order execution advances `ENGINE.lastMarkTime()`.
  3. Any trader attempting to submit an atomic protected open order via `commitOpenOrderWithProtection` reverts with `PletherOracle__PriceOutOfOrder` whenever `ENGINE.lastMarkTime() > acceptedTime` (or `PletherOracle__NoAcceptedBasket` after initial mark bootstrap via `initializeInitialMarkPrice`).

---

## Recommended Fix

In `PositionProtectionBook._freshProtectionObservation()`, inspect `oracle.getAcceptedBasketMark()` first. When `!accepted` or `acceptedTime < markTime` (the exact state that `_acceptedMarkInstalled()` allows because `ENGINE` already holds a newer fresh mark from order execution or initial mark bootstrap), use the already freshness-validated `(cachedMarkPrice, markTime)` from `ENGINE` instead of calling `oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh)`:

```diff
     function _freshProtectionObservation()
         private
         view
         returns (uint256 cachedMarkPrice, uint256 currentMarkPrice, uint64 currentPublishTime)
     {
         IPletherOracle oracle = _activeOracle();
         IPletherOracle.PolicySnapshot memory policy = oracle.getOrderExecutionPolicy(true);
         if (policy.oracleFrozen) {
             revert OrderRouter__ConditionalTriggerFrozen();
         }
         uint64 markTime = ENGINE.lastMarkTime();
         cachedMarkPrice = ENGINE.lastMarkPrice();
         if (
             cachedMarkPrice == 0 || markTime == 0
                 || OracleFreshnessPolicyLib.isStale(markTime, policy.maxStaleness, block.timestamp)
         ) {
             revert OrderRouter__ProtectionMarkTooStale();
         }
-        IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
-        currentMarkPrice = current.markPrice;
-        currentPublishTime = current.publishTime;
-        if (!_acceptedMarkInstalled()) {
-            revert PositionProtectionBook__OracleMarkAhead();
-        }
+        (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = oracle.getAcceptedBasketMark();
+        if (!accepted || acceptedTime < markTime) {
+            return (cachedMarkPrice, cachedMarkPrice, markTime);
+        }
+        if (acceptedTime > markTime || acceptedPrice != cachedMarkPrice) {
+            revert PositionProtectionBook__OracleMarkAhead();
+        }
+        IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
+        currentMarkPrice = current.markPrice;
+        currentPublishTime = current.publishTime;
     }
```


## Proof of Concept

# Proof of Concept — `PositionProtectionBook._freshProtectionObservation` Reverts After Historical Order Execution

## Test File

Save the following test file as `packages/perps/test/perps/spec/trader/PositionProtectionPostExecutionDoS.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../../shared/OracleSynchronizationTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

contract PositionProtectionPostExecutionDoSTest is OracleSynchronizationTestBase {

    uint256 internal constant LONG_TAKE_PROFIT = 90_000_000;
    uint256 internal constant LONG_STOP_LOSS = 110_000_000;

    function _protectionActions() internal view returns (IPositionProtectionActions) {
        return IPositionProtectionActions(address(router.positionProtectionBook()));
    }

    function test_HistoricalOrderExecutionAdvancesEngineMarkAndBricksAllPositionProtectionEntrypoints() public {
        IPositionProtectionActions protectionActions = _protectionActions();
        uint64 t0 = uint64(block.timestamp + 1);
        _advance(t0);

        // 1. A live mark refresh installs the accepted 6-feed basket at t0.
        router.updateMarkPrice{value: FEE}(_payload(t0, t0 - 1));
        (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        assertTrue(accepted);
        assertEq(acceptedPrice, 100_000_000);
        assertEq(acceptedTime, t0);
        assertEq(engine.lastMarkTime(), t0);

        // 2. ALICE commits an open order at t0, and a keeper executes it at t0 + 2 using the historical tick at t0 + 1.
        uint64 aliceOrderId = _commit(ALICE, 0);
        _advance(t0 + 2);
        OrderV3Types.ExecutionResult memory execResult =
            router.executeOrder{value: 2 * FEE}(aliceOrderId, _payload(t0 + 1, t0));
        assertEq(uint8(execResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));

        // Historical order execution advanced engine.lastMarkTime() to t0 + 1 while _acceptedBasket remains at t0.
        assertEq(engine.lastMarkTime(), t0 + 1);
        (, , acceptedTime) = pletherOracle.getAcceptedBasketMark();
        assertEq(acceptedTime, t0);

        PositionProtectionTypes.PositionProtectionParams memory params = PositionProtectionTypes.PositionProtectionParams({
            takeProfitTriggerPrice: LONG_TAKE_PROFIT,
            stopLossTriggerPrice: LONG_STOP_LOSS,
            closeLimits: PositionProtectionTypes.CloseFinancialLimits({
                maxExecutionNotionalAssets: 0,
                maxGrossAccountDebitAssets: 0,
                maxActionChargeAssets: 0,
                maxExplicitFeesAssets: 0
            })
        });

        // 3. ALICE immediately attempts to attach TP/SL protection to her newly opened position.
        // Even though _acceptedMarkInstalled() is designed to return true when acceptedTime < engine.lastMarkTime(),
        // _freshProtectionObservation() calls oracle.getLatestPrice(MarkRefresh) first, which reverts with
        // PletherOracle__PriceOutOfOrder(t0, t0 + 1).
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, t0, t0 + 1)
        );
        protectionActions.createPositionProtection(params);

        // 4. BOB also cannot submit commitOpenOrderWithProtection while engine.lastMarkTime() > acceptedTime.
        OrderV3Types.OrderRequest memory bobOpenRequest = OrderV3Types.OrderRequest({
            clientOrderId: keccak256("bob-protected-open"),
            side: CfdTypes.Side.LONG,
            isClose: false,
            sizeDelta: 10_000e18,
            marginDelta: 1000e6,
            targetPrice: 100_000_000,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp + 60),
                executionWindowSeconds: 60,
                allowedExecutionModes: 7,
                maxPostLeverageBps: type(uint32).max,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint128).max,
                maxExecutionNotionalAssets: type(uint128).max,
                maxGrossAccountDebitAssets: type(uint128).max,
                maxActionChargeAssets: type(uint128).max,
                maxExplicitFeesAssets: type(uint128).max,
                maxPostPositionSize: type(uint128).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0
            })
        });

        vm.prank(BOB);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, t0, t0 + 1)
        );
        protectionActions.commitOpenOrderWithProtection(bobOpenRequest, params);
    }

    function test_ExistingArmedProtectionCannotBeReplacedAfterAnotherTraderOrderExecutes() public {
        IPositionProtectionActions protectionActions = _protectionActions();
        uint64 t0 = uint64(block.timestamp + 1);
        _advance(t0);

        // 1. ALICE opens a position and arms TP/SL protection when acceptedTime == engine.lastMarkTime() == t0 + 1.
        uint64 aliceOrderId = _commit(ALICE, 0);
        _advance(t0 + 1);
        router.executeOrder{value: 2 * FEE}(aliceOrderId, _payload(t0 + 1, t0));
        router.updateMarkPrice{value: FEE}(_payload(t0 + 1, t0));
        uint64 bobOrderId = _commit(BOB, 0);

        PositionProtectionTypes.PositionProtectionParams memory initialParams =
            PositionProtectionTypes.PositionProtectionParams({
                takeProfitTriggerPrice: LONG_TAKE_PROFIT,
                stopLossTriggerPrice: LONG_STOP_LOSS,
                closeLimits: PositionProtectionTypes.CloseFinancialLimits(0, 0, 0, 0)
            });

        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(initialParams);

        // 2. BOB's queued order is executed at t0 + 2, advancing engine.lastMarkTime() to t0 + 2.
        _advance(t0 + 3);
        router.executeOrder{value: 2 * FEE}(bobOrderId, _payload(t0 + 2, t0 + 1));
        assertEq(engine.lastMarkTime(), t0 + 2);

        // 3. ALICE attempts to update her Stop-Loss via replacePositionProtection, which reverts with
        // PletherOracle__PriceOutOfOrder(t0 + 1, t0 + 2).
        PositionProtectionTypes.PositionProtectionParams memory updatedParams =
            PositionProtectionTypes.PositionProtectionParams({
                takeProfitTriggerPrice: 88_000_000,
                stopLossTriggerPrice: 108_000_000,
                closeLimits: PositionProtectionTypes.CloseFinancialLimits(0, 0, 0, 0)
            });

        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, t0 + 1, t0 + 2)
        );
        protectionActions.replacePositionProtection(protectionId, updatedParams);
    }

}
```

## Command to Run

```bash
forge test --root packages/perps --match-path test/perps/spec/trader/PositionProtectionPostExecutionDoS.t.sol -vv
```

## Verbatim Output

```text
Ran 2 tests for test/perps/spec/trader/PositionProtectionPostExecutionDoS.t.sol:PositionProtectionPostExecutionDoSTest
[PASS] test_ExistingArmedProtectionCannotBeReplacedAfterAnotherTraderOrderExecutes() (gas: 7537995)
[PASS] test_HistoricalOrderExecutionAdvancesEngineMarkAndBricksAllPositionProtectionEntrypoints() (gas: 5870228)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 19.50ms (11.62ms CPU time)
```

