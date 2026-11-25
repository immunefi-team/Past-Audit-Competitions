# \#98516 [SC-Low] TP/SL creation and replacement revert after a historical fill because the MED-28 fix added a check that contradicts _acceptedMarkInstalled()

**Submitted on Nov 8th 2026 at 10:53:58 UTC by @hungviet276 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98516
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

**Brief/Intro**
After an order is filled through the normal keeper path, createPositionProtection, replacePositionProtection and commitOpenOrderWithProtection revert with PletherOracle__PriceOutOfOrder for every trader until someone pays for a separate live mark refresh. This happens whenever the fill’s historical price is newer than the last live refresh, which is the usual case when keepers execute orders promptly. The cause is that PositionProtectionBook._freshProtectionObservation now contains two checks that require opposite timestamp orders. The oracle getter added by the MED-28 fix (PR #124) needs the accepted oracle basket to be at least as new as the Engine mark, while _acceptedMarkInstalled() (PR #120), a few lines later in the same function, explicitly allows an older accepted basket. As a result, traders can’t arm or tighten a stop-loss right after a fill, and one trader’s fill blocks other traders from adding or changing TP/SL. No funds are lost directly.

**Vulnerability Details**
Since PR #120 the protocol keeps two price states:

the Oracle’s accepted live basket (_acceptedBasket), written only when a live update goes through _resolveCurrentBasket (PletherOracle.sol L795-799);
the Engine mark, which historical order fills also move forward through _updateEngineMark (OrderRouterExecutionSidecar.sol L1053-1061):
if (result.publishTime > engine_.lastMarkTime()) {
    engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime);
}

So after a fill priced from a tick newer than the last live refresh, engine.lastMarkTime() is ahead of the accepted basket. This is an expected state. _acceptedMarkInstalled() returns true when acceptedTime < markTime, and afterOrderTerminal (L598) relies on it to arm a parent order’s attached protection right after its historical fill.

_freshProtectionObservation (PositionProtectionBook.sol L862-886) is used by _create (L229), _replace (L295) and _stageAttached (L463), so it covers createPositionProtection, replacePositionProtection and commitOpenOrderWithProtection. After checking that the Engine mark is fresh, it does this:

IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh); // PR #124
currentMarkPrice = current.markPrice;
currentPublishTime = current.publishTime;
if (!_acceptedMarkInstalled()) { // PR #120
    revert PositionProtectionBook__OracleMarkAhead();
}

getLatestPrice builds a snapshot from the accepted basket with enforcePublishOrder = true (PletherOracle.sol L892-901):

if (snapshot.publishTime < markTime) {
    revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
}
if (snapshot.publishTime == markTime && engine.lastMarkPrice() != snapshot.markPrice) {
    revert PletherOracle__EqualTimeMarkMismatch(snapshot.publishTime, snapshot.markPrice, engine.lastMarkPrice());
}

And _acceptedMarkInstalled() (PositionProtectionBook.sol L888-895) returns:

return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());

The first check requires acceptedTime >= markTime. The second requires acceptedTime <= markTime. Together they only pass when the two timestamps are equal and the prices match. The acceptedTime < markTime branch of _acceptedMarkInstalled() can no longer be reached in this function, so any fill newer than the last live refresh blocks all three entrypoints.

Regression from the MED-28 fix
I checked the history of this function:

PR #120 (8191921) added the _acceptedMarkInstalled() check, which accepts an accepted basket older than the Engine mark.
PR #124 (eadc5c5), the MED-28 fix, added the getLatestPrice(MarkRefresh) call right above it, which rejects exactly that state.
MED-28 (“Trader can arm a new protection from a stale engine mark after oracle state already crossed it”) is about the oracle being ahead of the Engine. The team’s resolution note says the guard “rejects arming when the accepted oracle mark is ahead of, or inconsistent with, the Engine mark”. The new call also blocks the opposite case, where the Engine is ahead because of a normal historical fill. That state is valid by the Book’s own _acceptedMarkInstalled() definition, and the Engine mark in it is newer than anything getLatestPrice can return, because getLatestPrice only replays the stored accepted basket (_acceptedLiveBasket, PletherOracle.sol L705-726).

This is not a staleness problem
Both prices are fresh in this state. The Engine mark staleness is checked first in the same function (L873-878), and the accepted basket staleness is checked inside _acceptedLiveBasket (PletherOracle.sol L720) before the publish-order check. The PoC also asserts that both timestamps are within the staleness limit and that the revert comes from getLatestPrice, not from margin, bounty or policy checks.

When it happens
The state is reached when a historical fill moves the Engine mark past the accepted basket, meaning no live refresh with a publish time at or after the fill’s tick landed between the commit and the fill. It doesn’t happen after every fill (if a newer live refresh lands first, the two are in sync), but it is the normal case when a keeper executes an order within a few seconds of its commit. The block lasts until the next live refresh.

Calling getLatestPrice with no update data doesn’t help. The only way out is a separate, fee-paying router.updateMarkPrice with fresh signed Pyth data. No entrypoint combines that refresh with creating or replacing protection, and in an active queue the next fill can move the Engine ahead again before the trader’s transaction lands.

Suggested fix
When the Engine mark is newer than the accepted basket, it is already the newest signed on-chain observation, so validate against it instead of reverting:

-        IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
-        currentMarkPrice = current.markPrice;
-        currentPublishTime = current.publishTime;
+        (bool acceptedExists,, uint64 acceptedTime) = oracle.getAcceptedBasketMark();
+        if (acceptedExists && acceptedTime < markTime) {
+            currentMarkPrice = cachedMarkPrice;
+            currentPublishTime = markTime;
+        } else {
+            IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
+            currentMarkPrice = current.markPrice;
+            currentPublishTime = current.publishTime;
+        }

This keeps everything else in place: the frozen-market check, the Engine staleness check, the _acceptedMarkInstalled() guard, the Oracle’s publish-order guard and the TP/SL geometry validation. When the accepted basket is equal or ahead, it still uses getLatestPrice as the MED-28 fix intended.

I checked this change locally. All 345 existing tests in test/perps/spec/trader/PositionProtection.t.sol and test/perps/spec/oracle/*.t.sol still pass. I also wrote three regression tests, included at the end of the PoC section, that expect create, open-with-protection and replace to succeed after a historical fill. They fail on the current code with PriceOutOfOrder and pass with the fix.

**Impact Details**
Take-profit and stop-loss are a user-facing protection feature of the perps market. With this bug:

Right after a normal fill, traders can’t arm or tighten protection until someone pays for a live refresh. This is when they most need it, right after opening, with an initial margin of 0.2% and a maintenance margin of 0.1%.
The effect crosses accounts. In the PoC, Alice’s fill blocks Bob’s commitOpenOrderWithProtection, and Bob’s fill blocks Alice from replacing her existing TP/SL. Neither trader did anything wrong.
commitOpenOrderWithProtection reverts as a whole, so nobody can open a position with attached TP/SL in this state.
No funds are lost directly, and the user can work around it by paying for an extra live update first. That is why I’m reporting it as Low: “Contract fails to deliver promised returns, but doesn’t lose value”.

**References**
PositionProtectionBook.sol: _freshProtectionObservation
PositionProtectionBook.sol: _acceptedMarkInstalled
PositionProtectionBook.sol: afterOrderTerminal relies on accepted < engine
PletherOracle.sol: publish-order check
PletherOracle.sol: accepted basket write
OrderRouterExecutionSidecar.sol: _updateEngineMark
PR #120 commit
PR #124 commit (MED-28 fix)


## Proof of Concept

**Proof of Concept**
The PoC was run against commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. No contract in scope needs to be modified.

The test extends the repository’s OracleSynchronizationTestBase, which deploys the production Oracle, Engine, pool and PositionProtectionBook, with a mocked Pyth. In that base, router is the repo’s LegacyOrderRouterHarness, which inherits OrderRouter unchanged and only adds a test-only scalar commitOrder adapter. The PoC doesn’t use that adapter. It commits orders through the production V3 commitOrder(OrderRequest) (called via IPerpsTraderActions). The other calls are production functions: updateMarkPrice, executeOrder, createPositionProtection, replacePositionProtection and commitOpenOrderWithProtection.

Steps
1. Check out the repository at commit a82d4fe.
2. Save the PoC source below as packages/perps/test/perps/spec/oracle/ProtectionAfterHistoricalFillPoC.t.sol.
3. Run:
cd packages/perps
FOUNDRY_VIA_IR=false forge test --match-path test/perps/spec/oracle/ProtectionAfterHistoricalFillPoC.t.sol -vv

Result from my local run:

[PASS] test_Control_CreateSucceedsAfterExtraLiveRefresh()
[PASS] test_PoC_CreateProtectionRevertsAfterHistoricalFill()
[PASS] test_PoC_OtherTradersOpenWithProtectionRevertsAfterAnotherFill()
[PASS] test_PoC_ReplaceProtectionRevertsAfterAnotherTradersFill()
Suite result: ok. 4 passed; 0 failed; 0 skipped

Scenario
Shared setup (_openAfterLiveRefresh):

1. At time t, a keeper posts an ordinary live mark refresh. The accepted basket and the Engine mark are both at t.
2. Alice commits a LONG order with commitOrder(OrderRequest). At t+2, a keeper executes it through router.executeOrder using the first tick after her commit (t+1).
3. The Engine mark is now at t+1, and the accepted basket is still at t.
4. The test asserts that the market is not frozen and that both timestamps are within the staleness limit, so both observations are valid and fresh.
5. It also checks that pletherOracle.getLatestPrice(MarkRefresh) itself reverts with PriceOutOfOrder(t, t+1). That is the call the MED-28 fix added to _freshProtectionObservation.
What the tests show
CreateProtectionRevertsAfterHistoricalFill: Alice’s createPositionProtection reverts with PriceOutOfOrder(t, t+1) immediately after her fill (same timestamp, t+2). It still reverts at t+50 when no live refresh has happened in between.
OtherTradersOpenWithProtectionRevertsAfterAnotherFill: Alice’s fill blocks Bob. His commitOpenOrderWithProtection reverts with the same error. After a separate paid router.updateMarkPrice, the identical request succeeds.
ReplaceProtectionRevertsAfterAnotherTradersFill: Alice arms TP/SL while the marks are in sync. Bob’s ordinary fill moves the Engine mark to t+3, while the accepted basket stays at t+2. Alice’s replacePositionProtection, which tightens her stop-loss, reverts with PriceOutOfOrder(t+2, t+3). After a separate paid live refresh, the same replacement succeeds.
Control_CreateSucceedsAfterExtraLiveRefresh: the same state plus one extra live refresh lets createPositionProtection succeed, which isolates the cause.
These four tests reproduce the bug on the current code. They are not a check of the fix: with the fix applied, the three vm.expectRevert tests fail with “next call did not revert as expected”. To check the fix, I use the separate regression tests at the end of this section.

PoC source
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../../shared/OracleSynchronizationTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";

/// @notice PoC: a historical order fill advances the Engine mark but not the Oracle's accepted live basket.
///         PositionProtectionBook._freshProtectionObservation then calls getLatestPrice(MarkRefresh), which reverts
///         PriceOutOfOrder(acceptedTime, engineMarkTime), so creating or replacing TP/SL fails until someone posts a
///         live update. Orders are committed through the production V3 `commitOrder(OrderRequest)` entrypoint.
contract ProtectionAfterHistoricalFillPoC is OracleSynchronizationTestBase {

    function _params() private pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.takeProfitTriggerPrice = 90_000_000; // LONG profits when the basket falls
        params.stopLossTriggerPrice = 110_000_000;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _book() private view returns (IPositionProtectionActions) {
        return IPositionProtectionActions(address(router.positionProtectionBook()));
    }

    /// Plain V3 LONG open request with permissive bounds.
    function _request(
        uint256 clientId
    ) private view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(clientId);
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = 10_000e18;
        request.marginDelta = 1000e6;
        request.targetPrice = 1;
        request.bounds = OrderV3Types.ExecutionBounds({
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

    /// Commits through the production V3 entrypoint (not the test harness's scalar adapter).
    function _commitV3(
        address trader,
        uint256 clientId
    ) private returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _request(clientId);
        vm.prank(trader);
        orderId = IPerpsTraderActions(address(router)).commitOrder(request);
    }

    /// Live refresh at t, Alice's open fills from historical ticks at t+1 -> Engine mark (t+1) ahead of accepted (t).
    function _openAfterLiveRefresh() private returns (uint64 t) {
        t = uint64(block.timestamp + 1);
        _advance(t);
        router.updateMarkPrice{value: FEE}(_payload(t, t - 20)); // ordinary live mark refresh
        assertEq(engine.lastMarkTime(), t);

        uint64 id = _commitV3(ALICE, 1);
        _advance(t + 2);
        router.executeOrder{value: 2 * FEE}(id, _payload(t + 1, t)); // ordinary keeper execution
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(id)), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(engine.lastMarkTime(), t + 1, "historical fill advanced the Engine mark");
        (bool accepted,, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        assertTrue(accepted);
        assertEq(acceptedTime, t, "accepted live basket is still the older refresh");

        // Both observations are valid and fresh: the market is open and neither is older than the staleness limit.
        IPletherOracle.PolicySnapshot memory policy = pletherOracle.getOrderExecutionPolicy(true);
        assertFalse(policy.oracleFrozen);
        assertLe(block.timestamp - engine.lastMarkTime(), policy.maxStaleness, "engine mark fresh");
        assertLe(block.timestamp - acceptedTime, policy.maxStaleness, "accepted basket fresh");

        // The revert comes from the Oracle getter that the MED-28 fix added, not from margin, bounty or policy checks.
        vm.expectRevert(abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, t, t + 1));
        pletherOracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
    }

    function test_PoC_CreateProtectionRevertsAfterHistoricalFill() public {
        uint64 t = _openAfterLiveRefresh();
        IPositionProtectionActions book = _book();

        // Immediately after her fill (same timestamp, t+2), Alice tries to add TP/SL.
        vm.prank(ALICE);
        vm.expectRevert(abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, t, t + 1));
        book.createPositionProtection(_params());

        // Still blocked at t+50 when no live refresh has happened in between.
        _advance(t + 50);
        vm.prank(ALICE);
        vm.expectRevert(abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, t, t + 1));
        book.createPositionProtection(_params());
    }

    /// Cross-account effect: ALICE's ordinary fill also blocks BOB from opening a position with attached TP/SL.
    function test_PoC_OtherTradersOpenWithProtectionRevertsAfterAnotherFill() public {
        uint64 t = _openAfterLiveRefresh();
        IPositionProtectionActions book = _book();
        OrderV3Types.OrderRequest memory request = _request(77);

        vm.prank(BOB);
        vm.expectRevert(abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, t, t + 1));
        book.commitOpenOrderWithProtection(request, _params());

        // After someone pays for a live refresh, the identical request is accepted.
        router.updateMarkPrice{value: FEE}(_payload(t + 2, t + 1));
        vm.prank(BOB);
        (uint64 parentId, uint64 protectionId) = book.commitOpenOrderWithProtection(request, _params());
        assertGt(parentId, 0);
        assertGt(protectionId, 0);
    }

    /// ALICE has armed TP/SL while marks are in sync. BOB's ordinary fill moves the Engine mark ahead, and ALICE can no
    /// longer replace (tighten) her TP/SL until someone pays for a live refresh.
    function test_PoC_ReplaceProtectionRevertsAfterAnotherTradersFill() public {
        uint64 t = _openAfterLiveRefresh();
        IPositionProtectionActions book = _book();

        router.updateMarkPrice{value: FEE}(_payload(t + 2, t + 1)); // marks in sync at t+2
        vm.prank(ALICE);
        uint64 protectionId = book.createPositionProtection(_params());

        uint64 bobId = _commitV3(BOB, 2);
        _advance(t + 4);
        router.executeOrder{value: 2 * FEE}(bobId, _payload(t + 3, t + 2)); // BOB's ordinary fill
        assertEq(engine.lastMarkTime(), t + 3);

        PositionProtectionTypes.PositionProtectionParams memory tighter = _params();
        tighter.stopLossTriggerPrice = 105_000_000;

        vm.prank(ALICE);
        vm.expectRevert(abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, t + 2, t + 3));
        book.replacePositionProtection(protectionId, tighter);

        // After a separate paid live refresh, the identical replacement succeeds.
        router.updateMarkPrice{value: FEE}(_payload(t + 4, t + 3));
        vm.prank(ALICE);
        book.replacePositionProtection(protectionId, tighter);
    }

    /// Control: identical state, but someone pays for a live refresh first -> creation succeeds.
    function test_Control_CreateSucceedsAfterExtraLiveRefresh() public {
        uint64 t = _openAfterLiveRefresh();
        router.updateMarkPrice{value: FEE}(_payload(t + 2, t + 1));
        IPositionProtectionActions book = _book();
        vm.prank(ALICE);
        uint64 protectionId = book.createPositionProtection(_params());
        assertGt(protectionId, 0);
    }

}

Fix verification (regression tests)
ProtectionAfterHistoricalFillFixCheck.t.sol uses the same scenarios, but it expects create, open-with-protection and replace to succeed after a historical fill. I ran it twice: once on the current code, and once with only PositionProtectionBook.sol changed as in the suggested fix.

cd packages/perps
FOUNDRY_VIA_IR=false forge test --match-path test/perps/spec/oracle/ProtectionAfterHistoricalFillFixCheck.t.sol -vv

On the current code (a82d4fe), all three fail with the bug:

[FAIL: PletherOracle__PriceOutOfOrder(1709794801, 1709794802)] test_Fix_CreateSucceedsAfterHistoricalFill()
[FAIL: PletherOracle__PriceOutOfOrder(1709794801, 1709794802)] test_Fix_OpenWithProtectionSucceedsAfterAnotherFill()
[FAIL: PletherOracle__PriceOutOfOrder(1709794803, 1709794804)] test_Fix_ReplaceSucceedsAfterAnotherTradersFill()

With the suggested fix, all three pass:

[PASS] test_Fix_CreateSucceedsAfterHistoricalFill()
[PASS] test_Fix_OpenWithProtectionSucceedsAfterAnotherFill()
[PASS] test_Fix_ReplaceSucceedsAfterAnotherTradersFill()
Suite result: ok. 3 passed; 0 failed; 0 skipped

// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../../shared/OracleSynchronizationTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";

/// @notice Regression tests for the suggested fix: the same scenarios as ProtectionAfterHistoricalFillPoC, but they
///         expect create / open-with-protection / replace to SUCCEED after a historical fill. They fail on the
///         unpatched code (PriceOutOfOrder) and pass once _freshProtectionObservation uses the Engine mark when
///         the accepted basket is older.
contract ProtectionAfterHistoricalFillFixCheck is OracleSynchronizationTestBase {

    function _params() private pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.takeProfitTriggerPrice = 90_000_000;
        params.stopLossTriggerPrice = 110_000_000;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _book() private view returns (IPositionProtectionActions) {
        return IPositionProtectionActions(address(router.positionProtectionBook()));
    }

    function _request(
        uint256 clientId
    ) private view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(clientId);
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = 10_000e18;
        request.marginDelta = 1000e6;
        request.targetPrice = 1;
        request.bounds = OrderV3Types.ExecutionBounds({
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

    function _commitV3(
        address trader,
        uint256 clientId
    ) private returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _request(clientId);
        vm.prank(trader);
        orderId = IPerpsTraderActions(address(router)).commitOrder(request);
    }

    function _openAfterLiveRefresh() private returns (uint64 t) {
        t = uint64(block.timestamp + 1);
        _advance(t);
        router.updateMarkPrice{value: FEE}(_payload(t, t - 20));
        uint64 id = _commitV3(ALICE, 1);
        _advance(t + 2);
        router.executeOrder{value: 2 * FEE}(id, _payload(t + 1, t));
        assertEq(engine.lastMarkTime(), t + 1);
        (,, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        assertEq(acceptedTime, t);
    }

    function test_Fix_CreateSucceedsAfterHistoricalFill() public {
        _openAfterLiveRefresh();
        IPositionProtectionActions book = _book();
        vm.prank(ALICE);
        uint64 protectionId = book.createPositionProtection(_params());
        assertGt(protectionId, 0);
    }

    function test_Fix_OpenWithProtectionSucceedsAfterAnotherFill() public {
        _openAfterLiveRefresh();
        IPositionProtectionActions book = _book();
        OrderV3Types.OrderRequest memory request = _request(77);
        vm.prank(BOB);
        (uint64 parentId, uint64 protectionId) = book.commitOpenOrderWithProtection(request, _params());
        assertGt(parentId, 0);
        assertGt(protectionId, 0);
    }

    function test_Fix_ReplaceSucceedsAfterAnotherTradersFill() public {
        uint64 t = _openAfterLiveRefresh();
        IPositionProtectionActions book = _book();
        router.updateMarkPrice{value: FEE}(_payload(t + 2, t + 1));
        vm.prank(ALICE);
        uint64 protectionId = book.createPositionProtection(_params());

        uint64 bobId = _commitV3(BOB, 2);
        _advance(t + 4);
        router.executeOrder{value: 2 * FEE}(bobId, _payload(t + 3, t + 2));
        assertEq(engine.lastMarkTime(), t + 3);

        PositionProtectionTypes.PositionProtectionParams memory tighter = _params();
        tighter.stopLossTriggerPrice = 105_000_000;
        vm.prank(ALICE);
        book.replacePositionProtection(protectionId, tighter);
    }

}


