# \#99833 [SC-Low] A trader withdraws free balance after their queued order is priced, so only the orders that win against LPs fill

**Submitted on Nov 15th 2026 at 12:39:24 UTC by @Audittens for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99833
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The project states that users [cannot cancel](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L366) queued orders, and that [binding order semantics](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L658-L659) stop traders from turning them into free options. Three kinds of order take funds from the account's free balance when they execute: an open that earns a price-impact rebate, a partial close, and an increase on the same side. That balance is not reserved at commit, so the trader can withdraw it at any time, and the order then fails at no cost. So the trader waits until the order's price is fixed and a later price is visible, keeps the orders the move favours, and kills the rest. LPs, the counterparty, pay for every fill the trader keeps.

## Vulnerability Details

An order is priced at the [first tick](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L614) published after the commit, called P1 here, and that price holds for as long as the order can still execute. At commit, the router [reserves only](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L27-L33) the execution bounty and the order's committed margin. The rest of the account's free balance stays withdrawable. When the owner withdraws, `validateWithdraw` looks at pending orders only for an account with no position, and only for an order that set a minimum balance, which is zero unless the trader sets it:

[packages/perps/src/CfdEngineSettlementSidecar.sol:261-272](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L261-L272)
```solidity
if (pos.size == 0) {
    (uint64 orderId, uint256 minimum, uint256 maxDebit) =
        OrderSettlementFloorLib.load(engine.orderRouter(), account);
    if (minimum != 0) {
        uint256 balance = IMarginClearinghouse(host.clearinghouse()).balanceAssets(account);
        if (balance < minimum || balance - minimum < maxDebit) {
            revert IOrderRouterErrors.OrderRouter__SettlementFloorWithdrawal(
                orderId, balance, minimum, maxDebit
            );
        }
    }
    return;
```

For an account with a position it checks only that the position stays healthy. A position holder cannot set that minimum either, because the router rejects it on any close and for any account with a position:

[packages/perps/src/router/OrderLiquidationBatchLogic.sol:307-312](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L307-L312)
```solidity
if (bounds.minPostSettlementBalanceAssets != 0) {
    (uint256 positionSize,,,,,,) = host.engine().positions(account);
    if (request.isClose || positionSize != 0 || head != 0) {
        revert OrderRouter__SettlementFloorUnsupported();
    }
}
```

So the owner can withdraw whatever free balance a queued order of theirs needs, and the order fails:

- **Open that earns a rebate.** Such an open needs a rebate reserve, and the planner [pays it](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L564-L567) from free balance first and takes the rest from the new position's margin. Admission previews the open with the account's whole free balance, so an open that is valid only with that extra balance is accepted. Once the balance is withdrawn, the reserve comes from the margin, equity falls to the initial margin, and the open [fails with](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L428-L437) `INSUFFICIENT_INITIAL_MARGIN`. The order ends for good, its margin returns to the account, and when the trader executes it itself the router [returns the bounty](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L336-L337) too.
- **Partial close.** Its charge (fee, price-impact charge and carry) is [taken partly](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1111-L1133) from free balance at execution. When part of it cannot be collected, a partial close [ends with](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L882-L885) `PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE`. The order fails for good, and closes reserve no bounty, so the trader pays nothing.
- **Increase.** An increase with no committed margin pays its fee and price-impact charge from free balance, and a shortfall [ends with](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L361-L364) `MARGIN_DRAINED_BY_FEES`. Commit does not test for it, because the planner [classes it](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L100-L102) as an execution-time error. To fill at all with no committed margin, the increase must fit in a surplus of the account's liquidation reserve, which a position has once its notional has fallen since it opened.

The two scenarios below cover the first two kinds: one for an account with no position and one for a position holder.

### Attack scenario: open

The setup uses the release risk parameters from `DEPLOYMENT.md`, with 1,000,000 USDG of Junior deposits and another trader long 300,000 USDG, so a short earns a rebate.

1. The trader finds the smallest margin at which a 200,000 USDG short is valid with no extra balance, using the public `previewOpenRevertCode`, and commits 10 USDG less: 670 USDG. It deposits that margin, the 0.20 USDG bounty and 1,000 USDG of extra free balance, and calls `OrderRouter.commitOrder` with no minimum balance. Admission passes because the extra balance is present.
2. P1 is published one second after the commit at 1.00 and fixes the open's price. The trader watches the later ticks.
3. If a later tick is 1% higher, which favours the short, the trader leaves the balance in place and executes the open with the P1 data through `OrderRouter.executeOrder`. The rebate reserve of 399 USDG comes from the extra balance, and the open fills at 1.00. The trader closes at 1.01 and ends with 3,501.22 USDG from 1,670.20 USDG deposited, a gain of 1,831.02 USDG, and the pool's cash falls 1,991.82 USDG.
4. If the later tick is 1% lower, the trader calls `MarginClearinghouse.withdrawMargin` for the 1,000 USDG and executes the open in one transaction. The project's SimpleAccount accepts [`executeBatch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa/SECURITY.md#L17) for this. The reserve now comes from the position's margin, the open fails with `INSUFFICIENT_INITIAL_MARGIN`, and the trader gets back every atom: margin, extra balance and bounty.
5. A keeper that executes the open at P1 before step 4 fills it as an ordinary trade. The trader can rule that out by withdrawing the 1,000 USDG right after the commit and depositing it back only when the later tick favours the short. Then a keeper that goes first only makes the open fail, and the trader loses the 0.20 USDG bounty. With no keeper first, the open fills at P1 after the deposit, with the same gain of 1,831.02 USDG.

### Attack scenario: partial close

1. The trader holds a long of 100,000 units at 1.00 with 400 USDG of margin, 210.1 USDG of pledge and 599.8 USDG of free balance, in a pool of about 1,002,000 USDG. Another account holds a short of 400,000 units, so the long is the smaller side.
2. The trader commits a partial close of 50,000 units. Closing the smaller side widens the gap between longs and shorts, so the charge is 182.28 USDG, more than the pledge the close releases.
3. P1 is published one second after the commit at 1.0005, which is bad for the long. Two seconds later Pyth shows 0.9995.
4. The trader withdraws the 599.8 USDG through `withdrawMargin`. The withdrawal passes, because the pledge still covers initial margin.
5. A keeper executes the close with P1. It fails with `PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE`, and the trader's position and balance are unchanged.
6. The trader deposits again, commits the same close, and it executes at 0.9995. The trader ends with 752 USDG instead of 702 USDG, and the pool's cash is 50 USDG lower. Had the price moved the other way, the trader would have let the first close fill at P1. The second close pays the same charge as the first, so any later price better than P1 is a gain.

## Impact Details

Direct theft of user funds. The trader takes LP deposits: after the order's price is fixed and a later price is visible, it lets the winning orders fill and makes the losing ones fail for free by withdrawing the balance they need. In the tests, a 1% move on a 200,000 USDG short gains the trader 1,831 USDG and costs the pool 1,992 USDG of cash, and a 10 bps move on a 50,000 USDG partial close costs the pool 50 USDG.

LPs lose deposited USDG, and the loss is final: the trader's gain is paid from the pool's cash through ordinary fills and closes. A cancelled order costs the trader nothing, so every order of these kinds is an option on the move after P1 that LPs write for free. The trader can repeat this with every such order, up to 5 pending orders per account.

Conditions:
- An open must shrink the gap between longs and shorts, so that it earns a rebate and needs a rebate reserve. The trader must have no position, and must leave the minimum balance at its default of zero.
- For a partial close, the charge must exceed the pledge it releases, usually a close that widens the gap between longs and shorts. The withdrawal must keep the position above initial margin at a fresh mark.
- For an increase, the cost must be positive and the increase must fit in a reserve surplus left by an earlier fall in the position's notional.
- The trader must act after a later tick is visible and before a keeper executes the order at P1. Nothing in the contracts limits the delay: P1 stays valid until the order's deadline, at most `maxExecutionWindowSeconds` after the commit, 60 seconds by default. A partial close pays keepers no bounty. The scale sets severity from the end state, and this race only lowers the reward.
- A kept open profits when the move after P1 exceeds its two 4 bps execution fees, about 8 bps in the test's setup. A cancelled and repeated partial close gains on any better price.

The [accounting specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L918-L931) offers the minimum balance as a way to stop owner withdrawals from making a flat account's open fail, and leaves orders without it withdrawable. It says the minimum ["prevents owner withdrawals from manufacturing a settlement-minimum failure"](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L930). The failures here are manufactured the same way, through other checks that the minimum does not cover. The specification does not describe a trader who leaves the minimum off on purpose to cancel a priced order and take LP funds. LPs cannot set it for the trader, and a position holder cannot set it at all.

The finding shows that the fix of SC-M4 is incomplete. SC-M4, on page 29 of the [pre-audit report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf), is the same free option: a losing order cancelled at no cost after the price is known, there through a refund that an LP withdrawal triggered. The fix fixed the price at the first tick after the commit and made failures terminal. Here the owner's own withdrawal makes the failure, so the free cancel is back.

It is not HIGH-1, on page 219 of the [Cecuro report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf), where a trader commits after a price is already published and is filled at it. Here P1 is published after the commit, and the same order fills or fails at the same P1 depending only on the withdrawal.

## Recommended Mitigation

Keep an owner's withdrawals from undoing what their queued orders rely on. Admit an open only on what stays locked: preview it at commit with the committed margin alone, not the account's free balance. For a partial close or an increase, reserve the charge at commit together with the order's committed margin. Alternatively, in `validateWithdraw`, reject a withdrawal that would leave free balance below what the account's pending orders can take from it, for flat accounts and position holders alike. Either way a withdrawal can no longer turn a priced order from valid to invalid.

## References

- [CfdEngineSettlementSidecar.sol:255-285](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L255-L285)
- [OrderCommitHandler.sol:21-36](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L21-L36)
- [OrderLiquidationBatchLogic.sol:307-312](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L307-L312)
- [CfdEnginePlanLib.sol:546-580](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L546-L580)
- [CfdEnginePlanLib.sol:428-437](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L428-L437)
- [CfdEnginePlanLib.sol:882-885](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L882-L885)
- [CfdEnginePlanLib.sol:361-364](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L361-L364)
- [MarginClearinghouseAccountingLib.sol:157-161](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarginClearinghouseAccountingLib.sol#L157-L161)

## Proof of Concept

The file has one test per attack scenario, both with the release risk parameters from `DEPLOYMENT.md` and 1,000,000 USDG of Junior deposits. Orders go through `OrderRouter.commitOrder` and `OrderRouter.executeOrder`, and withdrawals through `MarginClearinghouse.withdrawMargin`. Before the trader decides, each test moves two seconds past P1 and stores the later tick in Pyth. The orders are still executed with the P1 data.

The open test saves the state once P1 is fixed, plays out step 3 (the later tick is higher and the trader keeps the open), then reverts and plays out step 4 (the later tick is lower and the trader cancels it). It then plays out both outcomes of step 5. Amounts are logged in atoms (1 USDG = 1,000,000 atoms) to show that the cancellation returns every atom. The partial-close test plays out the ordinary fill at P1, then reverts and plays out the cancellation, so the two results can be compared.

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
import {console} from "forge-std/console.sol";

contract Test is BasePerpTest {

    address internal trader = makeAddr("trader");
    address internal otherTrader = makeAddr("otherTrader"); // holds the opposite side in both scenarios
    address internal keeper = makeAddr("keeper");

    uint256 internal constant SHORT_SIZE = 200_000e18; // 200,000 USDG at 1.00
    /// @dev 10 USDG below the smallest margin at which the short is valid with no extra balance,
    ///      found with the public `CfdEngineLens.previewOpenRevertCode`.
    uint256 internal constant MARGIN = 670_000_001;
    uint256 internal constant EXTRA_BALANCE = 1000e6;

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

    function test_openKeptOrCancelledAfterPriceIsKnown() public {
        // Setup: 1,000,000 USDG of Junior deposits and another trader long 300,000 USDG, so a short earns a rebate.
        _fundTrader(otherTrader, 50_000e6);
        uint64 longOpen = _commit(otherTrader, CfdTypes.Side.LONG, false, 300_000e18, 20_000e6);
        _execute(keeper, longOpen, _mockPythUpdateData(1e8), OrderV3Types.LifecycleStatus.Executed);

        // Step 1: the trader deposits 670 USDG of margin, the bounty and 1,000 USDG of extra free balance, and
        // commits a 200,000 USDG short. Admission passes because the extra balance is present.
        uint256 bounty = _quoteOpenOrderExecutionBountyAssets(SHORT_SIZE);
        uint256 deposited = MARGIN + bounty + EXTRA_BALANCE;
        _fundTrader(trader, deposited);
        uint64 shortOpen = _commit(trader, CfdTypes.Side.SHORT, false, SHORT_SIZE, MARGIN);
        uint256 poolCashBefore = settlementToken.balanceOf(address(pool));
        uint256 committed = vm.snapshotState();

        // Step 2: P1 is published one second after the commit at 1.00 and fixes the open's price.
        bytes[] memory p1 = _mockPythUpdateData(1e8);
        uint256 priced = vm.snapshotState();
        console.log("trader deposited (USDG atoms):               ", deposited);

        // Step 3: the later tick is 1% higher, so the trader leaves the balance in place and executes the open
        // with the P1 data. It fills at 1.00, and the trader closes at 1.01.
        _laterTick(1.01e8);
        (uint256 traderAfterKeep, uint256 poolCashAfterKeep) = _fillAndClose(shortOpen, p1, 1.01e8);
        console.log("Step 3, later tick 1% up, open kept:");
        console.log("  trader total after closing (USDG atoms):   ", traderAfterKeep);
        console.log("  pool cash lost (USDG atoms):               ", poolCashBefore - poolCashAfterKeep);
        assertGt(traderAfterKeep, deposited, "kept open pays");
        vm.revertToState(priced);

        // Step 4: the later tick is 1% lower instead, so the trader withdraws the 1,000 USDG and executes the
        // open itself. It fails, and the trader gets back every atom.
        _laterTick(0.99e8);
        vm.prank(trader);
        clearinghouse.withdrawMargin(EXTRA_BALANCE);
        _execute(trader, shortOpen, p1, OrderV3Types.LifecycleStatus.Failed);
        console.log("Step 4, later tick 1% down, open cancelled:");
        console.log("  trader total (USDG atoms):                 ", _traderTotal());
        console.log(
            "  pool cash change (USDG atoms):             ", settlementToken.balanceOf(address(pool)) - poolCashBefore
        );
        assertEq(_traderTotal(), deposited, "cancelled open costs nothing");
        vm.revertToState(committed);

        // Step 5: the trader withdraws the 1,000 USDG right after the commit instead. A keeper that executes
        // the open at P1 before the trader acts only makes it fail, and the trader loses the bounty.
        vm.prank(trader);
        clearinghouse.withdrawMargin(EXTRA_BALANCE);
        p1 = _mockPythUpdateData(1e8);
        priced = vm.snapshotState();
        _execute(keeper, shortOpen, p1, OrderV3Types.LifecycleStatus.Failed);
        console.log("Step 5, balance withdrawn after the commit, a keeper executes first:");
        console.log("  trader total (USDG atoms):                 ", _traderTotal());
        assertEq(_traderTotal(), deposited - bounty, "losing the race costs the bounty");
        vm.revertToState(priced);

        // When no keeper was first and the later tick is 1% higher, the trader deposits the 1,000 USDG back
        // and executes the open with the P1 data. It fills at 1.00 as in step 3.
        _laterTick(1.01e8);
        vm.startPrank(trader);
        settlementToken.approve(address(clearinghouse), EXTRA_BALANCE);
        clearinghouse.depositMargin(EXTRA_BALANCE);
        vm.stopPrank();
        (uint256 traderAfterRefill, uint256 poolCashAfterRefill) = _fillAndClose(shortOpen, p1, 1.01e8);
        console.log("Step 5, no keeper was first, later tick 1% up, balance deposited back:");
        console.log("  trader total after closing (USDG atoms):   ", traderAfterRefill);
        console.log("  pool cash lost (USDG atoms):               ", poolCashBefore - poolCashAfterRefill);
        assertGt(traderAfterRefill, deposited, "refilled open pays");
    }

    function test_partialCloseCancelledByWithdrawal() public {
        // Step 1: the trader holds a 100,000-unit long at 1.00 with 400 USDG of margin, in a pool of about
        // 1,002,000 USDG. Another account holds a 400,000-unit short, so the long is the smaller side.
        _fundTrader(trader, 1000e6);
        uint64 longOpen = _commit(trader, CfdTypes.Side.LONG, false, 100_000e18, 400e6);
        _execute(keeper, longOpen, _mockPythUpdateData(1e8), OrderV3Types.LifecycleStatus.Executed);
        _fundTrader(otherTrader, 10_000e6);
        uint64 shortOpen = _commit(otherTrader, CfdTypes.Side.SHORT, false, 400_000e18, 5000e6);
        _execute(keeper, shortOpen, _mockPythUpdateData(1e8), OrderV3Types.LifecycleStatus.Executed);

        // Step 2: the trader commits a partial close of 50,000 units.
        uint64 close = _commit(trader, CfdTypes.Side.LONG, true, 50_000e18, 0);

        // Step 3: P1 is published one second after the commit at 1.0005, which is bad for the long.
        bytes[] memory p1 = _mockPythUpdateData(1.0005e8);

        // For comparison: the trader leaves its balance in place and a keeper executes the close at P1.
        uint256 snapshot = vm.snapshotState();
        _execute(keeper, close, p1, OrderV3Types.LifecycleStatus.Executed);
        uint256 traderIfClosedAtP1 = _traderTotal();
        uint256 poolIfClosedAtP1 = settlementToken.balanceOf(address(pool));
        vm.revertToState(snapshot);

        // Step 4: the later tick is 0.9995, a better price for the long, so the trader withdraws its free balance.
        _laterTick(0.9995e8);
        uint256 free = clearinghouse.getFreeBuyingPowerAssets(trader);
        vm.prank(trader);
        clearinghouse.withdrawMargin(free);

        // Step 5: a keeper executes the close with the P1 data. It fails, and the position is unchanged.
        _execute(keeper, close, p1, OrderV3Types.LifecycleStatus.Failed);
        (uint256 size,,,,,,) = engine.positions(trader);
        assertEq(size, 100_000e18, "position unchanged");

        // Step 6: the trader deposits again, commits the same close, and it executes at 0.9995.
        vm.startPrank(trader);
        settlementToken.approve(address(clearinghouse), free);
        clearinghouse.depositMargin(free);
        vm.stopPrank();
        close = _commit(trader, CfdTypes.Side.LONG, true, 50_000e18, 0);
        _execute(keeper, close, _mockPythUpdateData(0.9995e8), OrderV3Types.LifecycleStatus.Executed);
        uint256 traderAfter = _traderTotal();
        uint256 poolAfter = settlementToken.balanceOf(address(pool));

        console.log("trader total, closed at P1 (USDG):           ", traderIfClosedAtP1 / 1e6);
        console.log("trader total, cancelled and re-closed (USDG):", traderAfter / 1e6);
        console.log("pool cash lost (USDG):                       ", (poolIfClosedAtP1 - poolAfter) / 1e6);

        assertGt(traderAfter, traderIfClosedAtP1, "trader gains");
        assertLt(poolAfter, poolIfClosedAtP1, "pool pays");
    }

    /// @dev Two seconds pass with the order still pending, and Pyth now shows `price`.
    function _laterTick(
        uint256 price
    ) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 2);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), int32(-8), vm.getBlockTimestamp());
    }

    /// @dev The trader executes its open with the P1 data and closes the short at `closePrice`.
    function _fillAndClose(
        uint64 shortOpen,
        bytes[] memory p1,
        uint256 closePrice
    ) internal returns (uint256 traderTotal, uint256 poolCash) {
        _execute(trader, shortOpen, p1, OrderV3Types.LifecycleStatus.Executed);
        (,, uint256 entryPrice,,,,) = engine.positions(trader);
        assertEq(entryPrice, 1e8, "filled at P1, not at the later tick");
        uint64 close = _commit(trader, CfdTypes.Side.SHORT, true, SHORT_SIZE, 0);
        _execute(keeper, close, _mockPythUpdateData(closePrice), OrderV3Types.LifecycleStatus.Executed);
        traderTotal = _traderTotal();
        poolCash = settlementToken.balanceOf(address(pool));
    }

    function _traderTotal() internal view returns (uint256) {
        return settlementToken.balanceOf(trader) + clearinghouse.balanceAssets(trader);
    }

    function _execute(
        address executor,
        uint64 orderId,
        bytes[] memory priceData,
        OrderV3Types.LifecycleStatus expected
    ) internal {
        vm.prank(executor);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, priceData);
        assertEq(uint8(result.status), uint8(expected), "execution outcome");
    }

    /// @dev The account commits through `OrderRouter.commitOrder`.
    function _commit(
        address account,
        CfdTypes.Side side,
        bool isClose,
        uint256 size,
        uint256 margin
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
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
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
[PASS] test_openKeptOrCancelledAfterPriceIsKnown() (gas: 9949123)
Logs:
  trader deposited (USDG atoms):                1670200001
  Step 3, later tick 1% up, open kept:
    trader total after closing (USDG atoms):    3501217494
    pool cash lost (USDG atoms):                1991817493
  Step 4, later tick 1% down, open cancelled:
    trader total (USDG atoms):                  1670200001
    pool cash change (USDG atoms):              0
  Step 5, balance withdrawn after the commit, a keeper executes first:
    trader total (USDG atoms):                  1670000001
  Step 5, no keeper was first, later tick 1% up, balance deposited back:
    trader total after closing (USDG atoms):    3501217494
    pool cash lost (USDG atoms):                1991817493
[PASS] test_partialCloseCancelledByWithdrawal() (gas: 8607202)
Logs:
  trader total, closed at P1 (USDG):            702
  trader total, cancelled and re-closed (USDG): 752
  pool cash lost (USDG):                        50

Suite result: ok. 2 passed; 0 failed; 0 skipped
```
