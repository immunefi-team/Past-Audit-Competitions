# \#99903 [SC-Low] A committed open can be ended by its own account after later prices are known, through a withdrawal of free balance

**Submitted on Nov 15th 2026 at 17:28:47 UTC by @Nyksx for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99903
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

The Router documents list "Binding intents" as an invariant (users cannot cancel queued orders once committed) and rely on it to keep queued intents from becoming free options. For an open that earns a VPI rebate, the position margin that the open leaves depends on the free settlement balance that the account holds at execution, and the withdrawal check holds that balance in place only for orders that carry a balance minimum. An account can therefore commit an open with its own `minPostPositionEquityAssets` limit and decide afterwards, by withdrawing its free balance, that the order ends as failed, at no cost when it executes the order itself. When a keeper's fill arrives some seconds after the pinned tick, a trader may keep the orders that later ticks favour and end the others, which may transfer value from LPs to the trader. 

## Finding Description

`packages/perps/SECURITY.md:366` lists the following Router invariant:

> | Binding intents | Users cannot cancel queued orders once committed |

`packages/perps/SECURITY.md:658-659` states the purpose:

> binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options,

`packages/perps/WHITEPAPER.md:1137-1138` repeats it ("Binding orders reduce the ability to treat a queue position as a free option, but they remove user cancellation after an order has entered the queue"), and `packages/perps/ACCOUNTING_SPEC.md:1266` states "Users cannot cancel a binding ordinary order."

Every order carries limits chosen by its account (`OrderV3Types.ExecutionBounds`). Two of them, `minPostPositionEquityAssets` and `maxPostLeverageBps`, are evaluated on the position margin that the open leaves. For an open that earns a VPI rebate, this margin depends on the free settlement balance of the account at execution. The open plan must fund the increase of the VPI rebate reserve, and it takes the amount from free settlement first and from the new position pledge only for the remainder.

`packages/perps/src/libraries/CfdEnginePlanLib.sol:561-579` (`_planOpenReserveFunding()`):

```solidity
        uint256 vpiReserveIncreaseAssets = delta.vpiRebateReserveAfterAssets > delta.vpiRebateReserveBeforeAssets
            ? delta.vpiRebateReserveAfterAssets - delta.vpiRebateReserveBeforeAssets
            : 0;
        // free settlement at execution funds the reserve first ...
        uint256 vpiReserveFromFreeAssets = vpiReserveIncreaseAssets < openCostPlan.resultingFreeSettlementAssets
            ? vpiReserveIncreaseAssets
            : openCostPlan.resultingFreeSettlementAssets;
        // ... and only the remainder is taken from the new pledge
        delta.vpiRebateReserveFromPledgeAssets = vpiReserveIncreaseAssets - vpiReserveFromFreeAssets;
        // ...
        delta.positionMarginAfterOpen =
            openCostPlan.resultingPositionMarginAssets - delta.vpiRebateReserveFromPledgeAssets - reserveIncreaseAssets;
```

The policy evaluator derives the post-open equity from this margin and compares it with the account's own limit.

`packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:367` and `:530-539`:

```solidity
        assessment.postPositionMarginAssets = delta.positionMarginAfterOpen;
        // ...
        uint256 normalizedEquityAssets =
            assessment.postPositionEquityAssets > 0 ? uint256(assessment.postPositionEquityAssets) : 0;
        if (assessment.postPositionEquityAssets < 0 || normalizedEquityAssets < bounds.minPostPositionEquityAssets) {
            revert ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation(
                OrderV3Types.ConstraintKind.PostPositionEquity,
                normalizedEquityAssets,
                bounds.minPostPositionEquityAssets
            );
        }
        _requireMaximum(OrderV3Types.ConstraintKind.PostLeverage, assessment.postLeverageBps, bounds.maxPostLeverageBps);
```

The committed margin and the open bounty are reserved at commit, but the remaining free settlement stays withdrawable while the order is pending. `MarginClearinghouse._withdraw()` calls `CfdEngine.checkWithdraw()` (`MarginClearinghouse.sol:464`, `CfdEngine.sol:730-747`), which calls the settlement sidecar. For an account without a position, the sidecar considers a pending order only if it carries a balance minimum.

`packages/perps/src/CfdEngineSettlementSidecar.sol:261-273` (`validateWithdraw()`):

```solidity
        if (pos.size == 0) {
            (uint64 orderId, uint256 minimum, uint256 maxDebit) =
                OrderSettlementFloorLib.load(engine.orderRouter(), account);
            // only minPostSettlementBalanceAssets of the pending open is protected
            if (minimum != 0) {
                uint256 balance = IMarginClearinghouse(host.clearinghouse()).balanceAssets(account);
                if (balance < minimum || balance - minimum < maxDebit) {
                    revert IOrderRouterErrors.OrderRouter__SettlementFloorWithdrawal(
                        orderId, balance, minimum, maxDebit
                    );
                }
            }
            // a pending open with an equity or leverage limit does not restrict the withdrawal
            return;
        }
```

For the balance minimum, the design keeps the committed state in place. `packages/perps/ACCOUNTING_SPEC.md:920-922` states that after commitment "settlement custody `S`, minimum `M` and maximum gross debit `G` must satisfy `S >= M && S - M >= G`. Every owner withdrawal applies the same predicate after its provisional debit". No corresponding rule covers the equity and leverage limits, although they depend on the same balance.

A failed own limit ends the order. The execution sidecar classifies the evaluator's `ConstraintViolation` revert as terminal for every constraint kind.

`packages/perps/src/OrderRouterExecutionSidecar.sol:879-893` (`_classifyTypedFailure()`):

```solidity
        if (
            selector == ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector
                && revertData.length == 100
        ) {
            uint256 constraint = _word(revertData, 4);
            if (constraint > 0 && constraint <= uint256(OrderV3Types.ConstraintKind.PostLeverage)) {
                // the order becomes Failed; it is not retried
                classification.terminal = true;
                classification.reason = OrderV3Types.TerminalReason.ConstraintViolation;
```

`executeOrder()` is permissionless (`OrderRouter.sol:240-247`), so the account can execute its own order. On a terminal outcome the reserved open bounty is paid to the executor, and it is released back to the account when the account is the executor.

`packages/perps/src/OrderRouter.sol:333-343` (`_settleTerminalBounty()`):

```solidity
        if (bountyAssets != 0) {
            settlement.bountyRecipient = bountyRecipient;
            settlement.bountyDisposition = OrderV3Types.BountyDisposition.Paid;
            // an order ended by its own account returns the bounty to that account
            if (bountyRecipient == order.account) {
                clearinghouse.releaseReservedExecutionBountyToSource(order.account, bountyAssets);
            } else {
                engine.creditBounty(
                    order.account, bountyRecipient, bountyAssets, accountingPrice, accountingPublishTime
                );
            }
        }
```

Together, these rules give the account the following choice after the commit:

1. The account holds some free settlement and commits an open on the lighter side of the book (an open that earns a VPI rebate). It sets `minPostPositionEquityAssets` between the position margin that the open leaves with the free settlement in place and the margin that it leaves without it. Both values follow from public state and can be simulated before the commit.
2. The order's price is pinned to the first Pyth tick after the commit. That tick is public from about `commitTime + 1`, and later ticks keep arriving while the order is pending.
3. If the later ticks favour the position, the account does nothing, and any executor fills the order at the pinned tick.
4. If the later ticks are adverse, the account withdraws its free settlement with `withdraw()`. The order now evaluates below its own limit. The account calls `executeOrder()` on its own order, which ends as `Failed` with `ConstraintViolation`; the committed margin is released and the bounty returns to the account. If a keeper executes the order first, the order ends in the same way and the keeper receives the bounty (0.20 USDG at the release parameters for orders of 2,000 USDG notional or more).
5. The account can deposit the withdrawn amount again and commit a new order.

The following conditions apply. The difference between the two margin values equals the part of the rebate reserve that moves from free settlement to the pledge, which is the smaller of the execution fee and the rebate (0.04% of the notional at the release execution fee when the rebate is larger). An open that pays VPI has no such dependence, so the behavior is limited to opens that reduce skew. Self-execution requires the order to be at the global FIFO head; behind another pending order.

- The SC Audit Studio pre-audit item SC-M4, "Risk Free Option in 60Sec Oracle Window" (`audits/plether-pre-audit-report.pdf`, pages 29-30), is marked "Fixed in Plether-Fi/plether-core#24". Its scenario includes a step in which the trader makes its own losing order fail after the price is known, by withdrawing junior LP funds so that the open fails the solvency check and its margin is refunded. The lever in this report is a different one: the account's own free settlement and its own execution limit, with no LP position involved. The current code holds free settlement in place for one own limit (the balance minimum) and not for the equity and leverage limits.

## Impact Explanation

In the proof of concept, a keeper fills every order 10 seconds after its commit, and the basket moves 0.20% in favour of a 100,000 USDG open, or 0.20% against it, after the pinned tick and before that fill. With a binding order the trader gains 114.89 USDG on the favourable move and loses 285.69 USDG on the adverse move, and the HousePool's assets change by −195.00 USDG and +205.41 USDG. When the trader ends the adverse order by a withdrawal, the adverse outcome becomes 0 USDG for the trader (0.20 USDG if the keeper executes the ended order) and 0 USDG for the pool, while the favourable outcome is unchanged. Over an even mix of the two moves, the trader's expected result per order changes from −85.40 USDG to +57.44 USDG, and the pool's from +5.20 USDG to −97.50 USDG. The amounts are measured at 100,000 USDG notional and grow with the order size, which is bounded by the existing skew because only opens that reduce skew earn a rebate. 

## Recommendation

Consider holding an account's free settlement in place while it has a pending open whose own limits depend on that balance, in the same way as the existing rule for the balance minimum.

## Proof of Concept

## Proof of Concept

The test deploys the production engine, HousePool, tranche vaults, `PletherOracle` and Router from `BasePerpTest` with the release risk parameters and the release Router overrides (`minOpenNotionalAssets = 1,000e6`, `adverseConfidenceMultiplierBps = 2,500`, 0.20 USDG trigger bounty, 5 pending orders), with the default 15-second settlement window. Only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocked. The basket uses the fixture's two equally weighted feeds, every tick carries a confidence of 0.01%, and the junior tranche holds a 20,000,000 USDG deposit. The test writes no storage and impersonates no protocol contract. The basket is at 0.80, and BOB holds a 2,000,000 USDG SHORT, so a LONG open reduces skew and earns a rebate. ALICE commits a 100,000 USDG LONG with 500 USDG margin at `c` through the production `commitOrder(OrderRequest)` with `minPostPositionEquityAssets = 380 USDG`. The first tick after the commit (c+1) is at 0.80, and the basket then moves by 0.20% or 0.10% (a LONG gains when the basket falls). A keeper executes pending orders at c+10 with the c+1 tick. BOB's open and ALICE's close use the scalar commit helper of the repository's Router test adapter (`LegacyOrderRouterHarness`, a subclass of `OrderRouter`).

1. `test_MarginAfterOpenDependsOnFreeBalance()`: without an own limit, the open leaves 400.00 USDG of position margin when ALICE keeps her free balance and 360.00 USDG when she withdraws it after the commit.
2. `test_BindingOrder_Adverse_KeeperFills()`: ALICE does nothing after an adverse move; the keeper fills at c+10 and ALICE closes at the new price.
3. `test_EndedOrder_Adverse_TraderExecutes_CostsNothing()`: after the same move ALICE withdraws her free balance at c+8 and executes her own order, which ends as `Failed` with `ConstraintViolation`.
4. `test_EndedOrder_Adverse_KeeperExecutes_CostsBounty()`: ALICE only withdraws at c+8; the keeper's execution at c+10 ends the order and earns the 0.20 USDG bounty.
5. `test_Favourable_KeeperFills()`: ALICE keeps her free balance; the keeper fills at c+10 and ALICE closes at the new price, for a 0.20% and a 0.10% move.
6. `test_Control_BalanceMinimumBlocksWithdrawal()`: for comparison, an order with a balance minimum blocks the withdrawal while it is pending.

| Run (100,000 USDG notional) | ALICE (USDG) | HousePool assets (USDG) |
| --- | --- | --- |
| Favourable 0.20%, keeper fills | +114.89 | −195.00 |
| Favourable 0.10%, keeper fills | +14.84 | −95.00 |
| Adverse 0.20%, binding order, keeper fills | −285.69 | +205.41 |
| Adverse 0.20%, ALICE ends her order | 0.00 | 0.00 |
| Adverse 0.20%, keeper ends the order after the withdrawal | −0.20 | 0.00 |

Command (from the repository root, forge 1.9.0, run against unmodified a82d4fef sources):

```
forge test --root packages/perps --match-path test/Submission_C11.t.sol -vv
```

Key output (excerpt: test results only, compiler warnings omitted; USDG atoms with 6 decimals):

```
Ran 6 tests for test/Submission_C11.t.sol:Submission_C11
[PASS] test_BindingOrder_Adverse_KeeperFills() (gas: 5618280)
Logs:
  binding order, adverse 0.20%
    ALICE net (USDG atoms): -285685579
    keeper net (USDG atoms): 200000
    pool change (USDG atoms): 205405577

[PASS] test_Control_BalanceMinimumBlocksWithdrawal() (gas: 1601167)
[PASS] test_EndedOrder_Adverse_KeeperExecutes_CostsBounty() (gas: 2642672)
Logs:
  order ended by the keeper, adverse 0.20%
    ALICE net (USDG atoms): -200000
    keeper net (USDG atoms): 200000
    pool change (USDG atoms): 0

[PASS] test_EndedOrder_Adverse_TraderExecutes_CostsNothing() (gas: 2626188)
Logs:
  order ended by ALICE, adverse 0.20%
    ALICE net (USDG atoms): 0
    keeper net (USDG atoms): 0
    pool change (USDG atoms): 0

[PASS] test_Favourable_KeeperFills() (gas: 11237835)
Logs:
  favourable 0.20%
    ALICE net (USDG atoms): 114885001
    keeper net (USDG atoms): 200000
    pool change (USDG atoms): -195004999
  favourable 0.10%
    ALICE net (USDG atoms): 14842500
    keeper net (USDG atoms): 200000
    pool change (USDG atoms): -95002499

[PASS] test_MarginAfterOpenDependsOnFreeBalance() (gas: 6708954)
Logs:
  position margin, free balance kept (USDG atoms): 400002500
  position margin, free balance withdrawn (USDG atoms): 360003500

Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 18.14ms (23.17ms CPU time)

Ran 1 test suite in 18.53ms (18.14ms CPU time): 6 tests passed, 0 failed, 0 skipped (6 total tests)
```

`packages/perps/test/Submission_C11.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @title A committed open is ended by its own account through a withdrawal of free balance
/// @dev Production contracts from `BasePerpTest` with the release risk parameters and the release Router overrides.
///      Only Pyth (MockPyth) and USDG (MockSettlementToken) are mocked. No storage write and no impersonation of a
///      protocol contract. The Router is the repository's test adapter around the production OrderRouter; its
///      scalar commit helper is used for BOB's open and for ALICE's close only.
///
///      Book: BOB holds a 2,000,000 USDG SHORT, so a LONG open reduces skew and earns a VPI rebate.
///      Scenario: ALICE commits a 100,000 USDG LONG with 500 USDG margin at c, through the production
///      `commitOrder(OrderRequest)`, with her own limit `minPostPositionEquityAssets = MIN_EQUITY`. The order's
///      price is the first tick after the commit (c+1, basket 0.80). The basket then moves; a LONG gains when it
///      falls. A keeper is active and executes every pending order 10 seconds after its commit.
///      - Binding order: ALICE does nothing; the keeper fills at c+10 at the c+1 tick; ALICE closes at the new price.
///      - Ended order: after an adverse move ALICE withdraws her free balance at c+8. The order then fails her
///        own limit and ends as Failed, when she executes it herself (the bounty returns to her) or when the
///        keeper executes it at c+10 (the keeper earns the 0.20 USDG bounty).
contract Submission_C11 is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);
    address internal constant KEEPER = address(0xC0FFEE);

    /// @dev Tuesday 2024-03-12 10:00 UTC (live market).
    uint256 internal constant T0 = 1_710_237_600;
    /// @dev Commit time of ALICE's open.
    uint256 internal constant C = T0 + 40;

    uint256 internal constant ENTRY = 0.8e8; // basket at the c+1 tick
    uint256 internal constant DOWN_20 = 0.7984e8; // -0.20%, favourable to the LONG
    uint256 internal constant DOWN_10 = 0.7992e8; // -0.10%
    uint256 internal constant UP_20 = 0.8016e8; // +0.20%, adverse to the LONG
    uint256 internal constant SIZE = 125_000e18; // 100,000 USDG notional at 0.80
    uint256 internal constant MARGIN = 500e6;
    uint256 internal constant OPEN_BOUNTY = 0.2e6;
    uint256 internal constant SKEW_SIZE = 2_500_000e18; // 2,000,000 USDG notional at 0.80
    uint256 internal constant SKEW_MARGIN = 20_000e6;
    uint256 internal constant UNLIMITED = type(uint256).max;
    /// @dev ALICE's own limit. It lies between the position margin that the open leaves with her free balance in
    ///      place and the margin it leaves without it (test_MarginAfterOpenDependsOnFreeBalance).
    uint256 internal constant MIN_EQUITY = 380e6;

    struct Outcome {
        uint8 status;
        uint8 reason;
        uint256 margin;
        int256 aliceNet;
        int256 keeperNet;
        int256 poolChange;
    }

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
        return 20_000_000e6;
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1000e6;
        config.adverseConfidenceMultiplierBps = 2500;
        config.positionProtectionTriggerBountyAssets = 0.2e6;
        config.maxPendingOrders = 5;
        _setRouterConfig(config);
        _fundTrader(ALICE, 10_000e6);
        _fundTrader(BOB, 100_000e6);

        // a keeper publishes the mark and fills BOB's SHORT
        _warpTo(T0);
        vm.prank(KEEPER);
        router.updateMarkPrice(_tick(ENTRY, T0));
        vm.prank(BOB);
        uint64 bobId = router.commitOrder(CfdTypes.Side.SHORT, SKEW_SIZE, SKEW_MARGIN, 0, false);
        _warpTo(T0 + 2);
        OrderV3Types.ExecutionResult memory r = _execute(KEEPER, bobId, _tick(ENTRY, T0 + 1));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "skew open executed");
        _warpTo(C);
    }

    // ================================================================== TESTS

    /// @dev Without an own limit the open fills in both cases, but the position margin it leaves depends on the
    ///      free balance that the account holds at execution, not at commit.
    function test_MarginAfterOpenDependsOnFreeBalance() public {
        uint256 snap = vm.snapshotState();
        Outcome memory kept = _run(0, ENTRY, address(0), false);
        vm.revertToState(snap);
        Outcome memory withdrawn = _run(0, ENTRY, KEEPER, false);
        emit log_named_uint("position margin, free balance kept (USDG atoms)", kept.margin);
        emit log_named_uint("position margin, free balance withdrawn (USDG atoms)", withdrawn.margin);
        assertEq(kept.status, uint8(OrderV3Types.LifecycleStatus.Executed), "kept: executed");
        assertEq(withdrawn.status, uint8(OrderV3Types.LifecycleStatus.Executed), "withdrawn: executed");
        assertGt(kept.margin, MIN_EQUITY, "with the free balance the open clears the limit");
        assertLt(withdrawn.margin, MIN_EQUITY, "without the free balance the open is below the limit");
    }

    /// @dev Binding order, adverse 0.20% move: the keeper fills at c+10 at the c+1 tick and ALICE carries the loss.
    ///      This is also the result when ALICE only waits for the order's deadline.
    function test_BindingOrder_Adverse_KeeperFills() public {
        Outcome memory o = _run(MIN_EQUITY, UP_20, address(0), true);
        _log("binding order, adverse 0.20%", o);
        assertEq(o.status, uint8(OrderV3Types.LifecycleStatus.Executed), "filled by the keeper");
        assertLt(o.aliceNet, -200e6, "ALICE is bound to the adverse move");
        assertGt(o.poolChange, 100e6, "pool collects more than 100 USDG");
    }

    /// @dev Same order and move. ALICE withdraws her free balance at c+8 and executes her own order. It ends as
    ///      Failed on her own limit and the bounty returns to her.
    function test_EndedOrder_Adverse_TraderExecutes_CostsNothing() public {
        Outcome memory o = _run(MIN_EQUITY, UP_20, ALICE, false);
        _log("order ended by ALICE, adverse 0.20%", o);
        assertEq(o.status, uint8(OrderV3Types.LifecycleStatus.Failed), "order ended as failed");
        assertEq(o.reason, uint8(OrderV3Types.TerminalReason.ConstraintViolation), "ended by her own limit");
        assertEq(o.aliceNet, 0, "ending the order costs ALICE nothing");
        assertEq(o.poolChange, 0, "pool collects nothing from the adverse move");
    }

    /// @dev Same order and move. ALICE only withdraws at c+8. The keeper's execution at c+10 ends the order and
    ///      earns the open bounty.
    function test_EndedOrder_Adverse_KeeperExecutes_CostsBounty() public {
        Outcome memory o = _run(MIN_EQUITY, UP_20, KEEPER, false);
        _log("order ended by the keeper, adverse 0.20%", o);
        assertEq(o.status, uint8(OrderV3Types.LifecycleStatus.Failed), "order ended as failed");
        assertEq(o.reason, uint8(OrderV3Types.TerminalReason.ConstraintViolation), "ended by her own limit");
        assertEq(o.aliceNet, -int256(OPEN_BOUNTY), "ALICE pays the open bounty only");
        assertEq(o.keeperNet, int256(OPEN_BOUNTY), "keeper earns the open bounty");
        assertEq(o.poolChange, 0, "pool collects nothing from the adverse move");
    }

    /// @dev Favourable moves: ALICE keeps her free balance, the keeper fills at c+10 at the c+1 tick, and the pool
    ///      pays. This leg is the same with and without the withdrawal option.
    function test_Favourable_KeeperFills() public {
        uint256 snap = vm.snapshotState();
        Outcome memory o20 = _run(MIN_EQUITY, DOWN_20, address(0), true);
        vm.revertToState(snap);
        Outcome memory o10 = _run(MIN_EQUITY, DOWN_10, address(0), true);
        _log("favourable 0.20%", o20);
        _log("favourable 0.10%", o10);
        assertGt(o20.aliceNet, 100e6, "0.20%: ALICE nets more than 100 USDG");
        assertLt(o20.poolChange, -100e6, "0.20%: pool pays more than 100 USDG");
        assertGt(o10.aliceNet, 0, "0.10%: ALICE nets a profit after fees");
        assertLt(o10.poolChange, 0, "0.10%: pool pays");
    }

    /// @dev Existing guard, for comparison: when the order carries a balance minimum, a withdrawal that breaks
    ///      `balance - minimum >= maxGrossAccountDebit` reverts while the order is pending.
    function test_Control_BalanceMinimumBlocksWithdrawal() public {
        uint256 balance = clearinghouse.balanceAssets(ALICE);
        OrderV3Types.OrderRequest memory req = _request(0, balance - 200e6, 200e6);
        vm.prank(ALICE);
        router.commitOrder(req);
        _warpTo(C + 8);
        vm.prank(ALICE);
        vm.expectPartialRevert(IOrderRouterErrors.OrderRouter__SettlementFloorWithdrawal.selector);
        clearinghouse.withdraw(ALICE, 1e6);
    }

    // ================================================================== scenario

    /// @param minEquity ALICE's own minimum for the position equity after the open (0 = none)
    /// @param laterPrice basket price after the c+1 tick; ALICE closes a filled position at this price
    /// @param ender address(0): ALICE does nothing and the keeper fills at c+10. ALICE: she withdraws her free
    ///        balance at c+8 and executes her own order. KEEPER: she withdraws at c+8 and the keeper executes at c+10.
    /// @param closeAfter whether ALICE closes a filled position (committed at c+11, executed at c+13)
    function _run(
        uint256 minEquity,
        uint256 laterPrice,
        address ender,
        bool closeAfter
    ) internal returns (Outcome memory o) {
        uint256 a0 = _total(ALICE);
        uint256 k0 = clearinghouse.balanceAssets(KEEPER);
        uint256 p0 = pool.totalAssets();

        OrderV3Types.OrderRequest memory req = _request(minEquity, 0, UNLIMITED);
        vm.prank(ALICE);
        uint64 id = router.commitOrder(req);

        if (ender != address(0)) {
            _warpTo(C + 8);
            uint256 free = clearinghouse.getAccountAssetBuckets(ALICE).freeSettlementAssets;
            vm.prank(ALICE);
            clearinghouse.withdraw(ALICE, free);
        }
        if (ender != ALICE) {
            _warpTo(C + 10);
        }
        // every executor supplies the same first tick after the commit
        OrderV3Types.ExecutionResult memory r = _execute(ender == ALICE ? ALICE : KEEPER, id, _tick(ENTRY, C + 1));
        o.status = uint8(r.status);
        o.reason = uint8(r.terminalReason);
        (, o.margin,,,,,) = engine.positions(ALICE);

        if (closeAfter) {
            _warpTo(C + 11);
            vm.prank(ALICE);
            uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
            _warpTo(C + 13);
            r = _execute(KEEPER, closeId, _tick(laterPrice, C + 12));
            assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close executed");
        }
        o.aliceNet = int256(_total(ALICE)) - int256(a0);
        o.keeperNet = int256(clearinghouse.balanceAssets(KEEPER)) - int256(k0);
        o.poolChange = int256(pool.totalAssets()) - int256(p0);
    }

    /// @dev Production V3 request: LONG open with the 60-second maximum execution window and no price limit.
    function _request(
        uint256 minEquity,
        uint256 minBalance,
        uint256 maxDebit
    ) internal view returns (OrderV3Types.OrderRequest memory req) {
        req = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("C11", router.nextCommitId())),
            side: CfdTypes.Side.LONG,
            sizeDelta: SIZE,
            marginDelta: MARGIN,
            targetPrice: 1,
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
                executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: UNLIMITED,
                maxExecutionNotionalAssets: UNLIMITED,
                maxGrossAccountDebitAssets: maxDebit,
                maxActionChargeAssets: UNLIMITED,
                maxExplicitFeesAssets: UNLIMITED,
                maxPostPositionSize: UNLIMITED,
                minPostSettlementBalanceAssets: minBalance,
                minPostPositionEquityAssets: minEquity,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    // ================================================================== helpers

    /// @dev Pyth update data for one basket tick: both feeds at `price` with 0.01% confidence, published at `t`.
    function _tick(
        uint256 price,
        uint256 t
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ups = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ups[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice({
                    price: int64(uint64(price)),
                    conf: uint64(price / 10_000),
                    expo: -8,
                    publishTime: t,
                    prevPublishTime: t - 1
                })
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ups);
    }

    function _execute(
        address executor,
        uint64 id,
        bytes[] memory data
    ) internal returns (OrderV3Types.ExecutionResult memory r) {
        vm.prank(executor);
        r = router.executeOrder(id, data);
    }

    function _warpTo(
        uint256 t
    ) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(t);
    }

    function _log(
        string memory label,
        Outcome memory o
    ) internal {
        emit log_string(label);
        emit log_named_int("  ALICE net (USDG atoms)", o.aliceNet);
        emit log_named_int("  keeper net (USDG atoms)", o.keeperNet);
        emit log_named_int("  pool change (USDG atoms)", o.poolChange);
    }

    /// @dev Wallet plus clearinghouse balance.
    function _total(
        address a
    ) internal view returns (uint256) {
        return settlementToken.balanceOf(a) + clearinghouse.balanceAssets(a);
    }

}
```
