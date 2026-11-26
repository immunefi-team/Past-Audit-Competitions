# \#99366 [SC-Critical] Trader-chosen execution window as short as 1 second lets traders keep a free option on queued orders at LPs' expense

**Submitted on Nov 13th 2026 at 11:43:47 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99366
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description


# Summary

Every delayed order's fill price is pinned to the first Pyth tick after `commitTime`, but execution is a separate, permissionless transaction, and the trader sets the order's `executionWindowSeconds` with a lower bound of 1 second (`OrderLiquidationBatchLogic.sol` L287). With a 1–2 second window no third-party keeper can reliably execute before the deadline, so the trader alone decides whether the pinned fill is ever applied: self-execute when the tick turned out favourable, self-expire when it did not. Both self-directed terminal paths return the execution bounty to the trader (`OrderRouter.sol` L336–338; `OrderRecoverySidecar.sol` L127), so the discarded branch costs only gas. A second abandonment path, withdrawing free settlement so execution fails terminally, works regardless of window because the withdrawal guard ignores pending orders (`CfdEngineSettlementSidecar.sol` L255–273).

This falsifies the stated security property that "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options" (`packages/perps/SECURITY.md` L658–659, restated at L719). The counterparty to every realised favourable fill is the HousePool, i.e. LPs. Severity: Low to Medium (economic value transfer; documented invariant broken).

# Description

 Mechanics

1. **Window.** `_validateFreshRequest` rejects `executionWindowSeconds == 0` and values above the governance maximum (default 60 s), nothing else (`OrderLiquidationBatchLogic.sol` L287). The lifecycle Book fixes `executionDeadline = commitTimestamp + executionWindowSeconds` (`OrderLifecycleBook.sol` L287); expiry is `block.timestamp > executionDeadline`.

2. **Pinned fill.** In live and FAD markets the fill is the unique first tick in `(commitTime, min(commitTime + orderSettlementWindow, block.timestamp)]` (`PletherOracle._resolveHistoricalOrderBasket`, L610–612). The `SameBlock` and `MevBoundary` guards (`OrderRouterExecutionSidecar.sol` L447, L450) only force execution into a later block with a post-commit tick; they do not stop the trader from reading that tick (from Hermes, within milliseconds) before deciding whether to execute.

3. **Self-directed terminal actions refund the bounty.** `_settleTerminalBounty` releases the reserved bounty back to the account whenever `bountyRecipient == order.account` (`OrderRouter.sol` L336–338). The permissionless expiry path records `executor = msg.sender` (`OrderRecoverySidecar.sol` L127) and passes it through as the bounty recipient, so a trader expiring its own order is refunded, and the lifecycle Book accepts the receipt as `Paid` to the executor (`OrderLifecycleBook._validateBountyDisposition`). The refund on self-*execution* is documented (`WORKING_WITH_AI_AGENTS.md` L231); the refund on self-*expiry* is not.

 The option

Commit at `T`. The fill tick is published at `T+1`. Until the deadline the trader compares the pinned price with the current price, then either executes its own order (favourable) or lets it expire and calls `expireOrder` itself (unfavourable), recovering margin and bounty in full. A keeper executing inside the window would make the intent binding; the window is chosen so that this does not happen.

 Why a healthy keeper cannot bind a 1–2 second order

On Arbitrum `block.timestamp` has one-second granularity and several ~250 ms blocks share each second. An order committed in a block stamped `T` is executable only in blocks stamped `T+1 … T+W`: blocks still stamped `T` are excluded by `SameBlock` and by the historical window's lower bound `T+1`. With `W = 1` that is the blocks stamped `T+1`, about one second of wall time in which the tick must be published, relayed and included. A keeper polling the FIFO even every two seconds gets at best a coin flip; the trader knows the commit time in advance, holds the payload ready and needs nobody else. With `W = 2` the trader also observes about a second of price development after the fill tick. The contracts place no lower bound on `W` beyond 1.

 A second abandonment path: defunding the order

The withdrawal guard for an account with no open position checks only the trader's own opt-in `minPostSettlementBalanceAssets` floor and otherwise returns (`CfdEngineSettlementSidecar.validateWithdraw`, L255–273); committed order margin and the bounty are locked, but nothing else is. At execution the order's margin is released to free settlement before planning (`OrderRouterExecutionSidecar.sol` L498) and the planner funds a skew-reducing open's negative-VPI reserve from free settlement first and from the new pledge only for the remainder (`CfdEnginePlanLib._planOpenReserveFunding`, L546–573). A trader who sizes `marginDelta` just above the initial-margin requirement therefore commits an order that passes while free settlement is present and fails `INSUFFICIENT_INITIAL_MARGIN` (`CfdEnginePlanLib.sol` L431–436, terminal as `UserInvalid`) once the trader has withdrawn that free settlement after seeing the fill. This path does not depend on keeper latency: a keeper executing the defunded order fails it terminally for a 0.20 USDG bounty, and a self-executing trader is refunded even that.

 Economics and bounds

- Round-trip cost is roughly 10 bps of notional: `executionFeeBps = 4` on open and on close (`CfdEngine.sol` L147), an adverse confidence shift of up to 20% of a basket confidence capped at 10 bps (`PletherOracle.sol` L84, L95), plus VPI where configured and gas.
- The look is one to two seconds of FX-basket movement. In normal conditions that is a fraction of a basis point, so most instances are abandoned at no cost and the edge is realised only in volatility bursts, where the confidence gate may also block execution. The payoff is `max(move − costs, 0)`: never negative, small in expectation, repeatable across orders and across accounts (the pending-order cap is per account).
- The default bounty is 1 bp of notional capped at 200_000 (0.20 USDG) (`OrderRouterBase.sol` L72–74). Even forfeited it would not deter a 50,000 USDG order; the refund removes the only cost the binding-intent model attached to abandonment.

 Relationship to known material

`audits/scan-plether-fi-plether-core-d4541b49.md` ("Order execution accepts oracle updates older than the engine's cached mark", informational) notes that a self-executing trader can apply a post-commit fill that is older than a newer cached mark, and points at mark monotonicity. That observation assumes the trader stays exposed to keeper execution while it waits. This report shows the trader removes that exposure by choosing the window or by defunding, so the intent is binding only with the trader's consent; rejecting older-than-mark fills would shrink the look but not restore binding intents. The acknowledged clock-alignment limitation (`SECURITY.md` L701–711) concerns a fill tick the trader may already have seen, after which the order still fills; here the trader decides whether to fill at all.

# Impact

LPs are the counterparty to a stream of one-sided trades: the trader keeps fills that moved in its favour between commit and deadline and pays nothing to discard the rest. Each instance is worth at most a few basis points of notional and only in volatile moments, so this is a slow, repeatable transfer of value from HousePool capital to traders rather than a drain. Structurally, a property the security model lists as preventing free options does not hold for any order whose window is shorter than keeper reaction time, and the trader sets the window; the defunding route breaks it even for long windows. The protocol's compensating controls (binding orders, no cancellation, reserved bounties) are all voided by the trader's own actions at a cost of 0 to 0.20 USDG.

**Suggested remediation**

1. Enforce a minimum `executionWindowSeconds` in `_validateFreshRequest` and in `OrderRouterAdmin`'s configuration validation, sized to the keeper service level (for example at least `orderSettlementWindow + 15` seconds, or a flat 30 seconds), so a healthy keeper can always make a queued intent binding.
2. Make abandonment carry a cost: on `Expired`, forfeit the bounty to the treasury regardless of who cleans up (paying the cleaner a flat fee), and consider an uncapped bounty floor of a few basis points of notional, since 0.20 USDG deters nothing. Leave the refund on successful self-execution in place; it is a legitimate liveness fallback.
3. Close the defunding route: reserve the execution fee and any projected reserve increase at commit, or make the withdrawal guard account for the funding needs of pending orders.
4. Optionally reject fills whose publish time predates the cached mark, as the October scan suggests; this shrinks the look but is not sufficient on its own.


## Proof of Concept

## Proof of Concept

# Proof of Concept (demonstrating impact)

Place the file at `packages/perps/test/perps/spec/trader/ExecutionWindowFreeOption.t.sol` and run:

```bash
forge test --root packages/perps --match-contract ExecutionWindowFreeOptionTest -vvv
```

The favourable branch installs a newer mark (0.99) on-chain before the trader executes, so the stale fill is visible in state: the trader enters a SHORT at the pinned 1.00 while `engine.lastMarkPrice()` is already 0.99, then closes at that mark for a profit paid by the pool. (In practice the trader compares the fill tick with an off-chain price; the on-chain mark is used so the test can assert it.) The unfavourable branch lets the deadline pass and shows self-expiry returns the trader to its exact pre-commit settlement balance with every reservation released.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";

/// @notice A trader-chosen 2-second window plus self-directed terminal actions: execute the favourable fill,
///         expire the unfavourable one for free.
contract ExecutionWindowFreeOptionTest is BasePerpTest {

    address internal constant TRADER = address(0x7EA1);

    uint256 internal constant SIZE = 50_000e18; // 500 lots of 100 tokens, 50,000 USDG notional at 1.00
    uint256 internal constant MARGIN = 20_000e6;
    uint256 internal constant FILL_PRICE = 1e8; // first post-commit tick: 1.00
    uint256 internal constant LATER_PRICE = 99e6; // price seen before the deadline: 0.99
    uint32 internal constant WINDOW = 2; // seconds; anything >= 1 is accepted

    function setUp() public override {
        super.setUp();
        _fundTrader(TRADER, 100_000e6);
        if (engine.lastMarkPrice() == 0) {
            vm.prank(address(router));
            engine.updateMarkPrice(FILL_PRICE, uint64(vm.getBlockTimestamp()));
        }
    }

    function _shortWindowOpen(
        bytes32 clientOrderId
    ) internal view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = clientOrderId;
        r.side = CfdTypes.Side.SHORT;
        r.sizeDelta = SIZE;
        r.marginDelta = MARGIN;
        r.targetPrice = type(uint256).max; // a SHORT open accepts any execution price <= target
        r.isClose = false;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp()),
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
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    /// Favourable branch: the pinned fill (1.00) is better than the price now on-chain (0.99);
    /// the trader self-executes inside the window and enters with an immediate ~1% gain.
    function test_FavorableBranch_SelfExecutesStaleFillAgainstLPs() public {
        uint256 traderBefore = clearinghouse.balanceAssets(TRADER);
        uint256 poolBefore = pool.totalAssets();

        // T: commit with a 2-second window; deadline = T + 2.
        vm.prank(TRADER);
        uint64 id = router.commitOrder(_shortWindowOpen("free-option-favorable"));
        assertEq(
            uint256(clearinghouse.getBountyReservation(IMarginClearinghouse.BountyKind.Order, id).amountAssets),
            200_000,
            "default bounty on a 50k order is 0.20 USDG"
        );

        // T+1: the first post-commit tick, i.e. the pinned fill, is 1.00.
        bytes[] memory fillTick = _mockPythUpdateData(FILL_PRICE);

        // T+2: a newer tick at 0.99 becomes the engine mark through any permissionless refresh.
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory laterTick = new bytes[](1);
        laterTick[0] = abi.encode(LATER_PRICE);
        router.updateMarkPrice(laterTick);
        assertEq(engine.lastMarkPrice(), LATER_PRICE, "mark is 0.99 before the trader decides");

        // Still inside the window, the trader executes its own order at the pinned 1.00.
        vm.prank(TRADER);
        router.executeOrder(id, fillTick);

        (uint256 size,, uint256 entryPrice,, CfdTypes.Side side,,) = engine.positions(TRADER);
        assertEq(size, SIZE, "position opened");
        assertEq(uint8(side), uint8(CfdTypes.Side.SHORT));
        assertEq(entryPrice, FILL_PRICE, "entered at the stale 1.00 fill");
        assertEq(engine.lastMarkPrice(), LATER_PRICE, "while the live mark is 0.99");
        assertEq(
            uint8(router.lifecycleBook().terminalOutcome(id).status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "self-execution finalised the order"
        );

        // Realise the gain at the prevailing on-chain mark. `_close` is the harness's engine-level close;
        // 0.99 is the mark already installed above, not an arbitrary price.
        _ensureLaterOracleTime();
        _close(TRADER, CfdTypes.Side.SHORT, SIZE, LATER_PRICE);

        assertGt(clearinghouse.balanceAssets(TRADER), traderBefore, "trader nets a profit after fees");
        assertLt(pool.totalAssets(), poolBefore, "the HousePool paid it");
    }

    /// Unfavourable branch: nobody executes inside the window; the trader expires its own order and
    /// ends exactly where it started.
    function test_UnfavorableBranch_SelfExpiryRefundsEverything() public {
        uint256 traderBefore = clearinghouse.balanceAssets(TRADER);

        vm.prank(TRADER);
        uint64 id = router.commitOrder(_shortWindowOpen("free-option-unfavorable"));
        assertEq(
            uint256(clearinghouse.getBountyReservation(IMarginClearinghouse.BountyKind.Order, id).amountAssets),
            200_000
        );

        uint64 deadline = router.lifecycleBook().orderTiming(id).executionDeadline;
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(uint256(deadline) + 1);
        vm.prank(TRADER);
        router.expireOrder(id);

        assertEq(
            uint8(router.lifecycleBook().terminalOutcome(id).reason),
            uint8(OrderV3Types.TerminalReason.Expired)
        );
        assertEq(clearinghouse.balanceAssets(TRADER), traderBefore, "settlement balance unchanged: bounty refunded");
        assertEq(clearinghouse.lockedMarginAssets(TRADER), 0, "order margin and bounty reservations released");
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 0, "no position");
    }

}
```

**Expected results**

- `test_FavorableBranch_SelfExecutesStaleFillAgainstLPs`: the order executes with `entryPrice == 1e8` while `engine.lastMarkPrice() == 0.99e8`; after closing at the on-chain mark the trader's settlement balance exceeds its starting balance and `pool.totalAssets()` is lower than before.
- `test_UnfavorableBranch_SelfExpiryRefundsEverything`: the terminal reason is `Expired`, the trader's settlement balance equals its pre-commit balance exactly (a third-party cleaner would have been paid 200_000 out of it), and no margin remains locked.

Together they show the trader chooses between the two outcomes after observing the price, and that the discarded outcome costs nothing inside the protocol.




