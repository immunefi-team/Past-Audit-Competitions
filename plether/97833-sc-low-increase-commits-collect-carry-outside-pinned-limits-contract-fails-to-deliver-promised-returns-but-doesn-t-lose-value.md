# \#97833 [SC-Low] Increase commits collect carry outside pinned limits: Contract fails to deliver promised returns, but doesn't lose value

**Submitted on Nov 3rd 2026 at 13:45:15 UTC by @c0mpos3r for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97833
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Every V3 order pins inclusive financial limits, and `maxGrossAccountDebitAssets` "covers settlement debit, trader-claim consumption, and the reserved bounty" (`OrderV3Types.sol:131-136`). An increase's commit locks its bounty and margin through Clearinghouse hooks that first collect the position's accrued carry, but the open commitment records that collection as zero and the evaluator adds commitment effects to closes only. The carry therefore escapes both the gross-debit and the action-charge limit, while a close collecting the same carry is checked against both. In the PoC an increase pinned to a 3.015817 USDG gross-debit limit executes after its commit took 31.134060 USDG of carry.

## Vulnerability Details

1. The commit collects carry. Every open calls `lockReservedSettlement` from `_reserveExecutionBounty` (`router/OrderReservationAccounting.sol:120-128`), because the bounty is clamped up to a minimum (`router/OrderLiquidationBatchLogic.sol:448-454`) that cannot be configured to zero (`OrderRouterAdmin.sol:381-388`). That function and `reserveCommittedOrderMargin` checkpoint carry first (`MarginClearinghouse.sol:822`, `:632`); for an account with a position this realizes the accrued carry through the Engine (`CfdEngine.sol:535-550`, `:1407-1449`).
2. The commitment records none of it. The open branch sets only `bountyFromFreeAssets` (`router/OrderReservationAccounting.sol:120-128`), so `recordCommitment` checks `0 + bounty` against `maxGrossAccountDebitAssets` and `0` against `maxActionChargeAssets` (`OrderLifecycleBook.sol:335-347`).
3. Execution does not see it. The evaluator adds the commitment for closes only (`CfdOrderPolicyEvaluator.sol:65-67`). Its open assessment does charge an increase's pending carry to both limits (`CfdOrderPolicyEvaluatorBase.sol:342-347`), but after the commit's checkpoint only the interval between commit and execution remains.

Closes use `reserveCloseOrderExecutionBounty`, which returns the carry it realizes (`CfdEngine.sol:592-625`), so the same amount is checked at commit and again at execution (`CfdOrderPolicyEvaluatorBase.sol:474-487`).

## Why this is not the documented design

Collecting carry at commit is documented (`INTERNAL_ARCHITECTURE_MAP.md:96`); leaving it outside the order's limits is not. Every fresh order pins "inclusive financial limits" (`SECURITY.md:368`), gross account debit "includes settlement value lost or charged" (`SECURITY.md:763-765`), and the accounting spec states that "financial bounds include the direct collection once" without exempting opens (`ACCOUNTING_SPEC.md:723`). The team tests this for closes (`test_CancelAndCommitClose_AccruedCarryCannotBypassGrossDebitBound` and its action-charge twin, `test/perps/spec/trader/PositionProtection.t.sol:1969`, `:1989`). The other custody limit, `minPostSettlementBalanceAssets`, is rejected for any account holding a position (`router/OrderLiquidationBatchLogic.sol:307-311`), so it cannot cover an increase. No report in `audits/` covers this; Cecuro LOW-34 (Acknowledged) concerns the carry that close assessments report, not open commits or what the limits count.

## Impact Details

In the PoC (release risk parameters and minimum open notional, `DEPLOYMENT.md:130-148`), a trader holds a 1,000-lot LONG with 10,000 USDG margin for 28 days, then commits a 20-lot increase.

- Pinned to limits equal to its own execution assessment (gross debit 3.015817, action charge 2.815817 USDG), the increase executes. Its commit realized 31.134060 USDG of carry, the commitment recorded 0, and the account was debited 34.149877 USDG in total. Either limit one atom lower fails at execution, so both bind.
- From the same state, a 20-lot partial close pinned to the increase's exact limits reverts at commit with `CommitmentBoundExceeded` on each limit, citing the same 31.134060 USDG of carry as the actual value.

The carry is accrued debt that the account owes either way, so no value is lost. The contract fails to deliver the limit it promises per order: `SECURITY.md:368` lists inclusive financial limits as the order's bounded authority, and the close path enforces the same limit on the same carry. The amount the limits miss is whatever carry accrued since the account's last checkpoint.

## Recommended fix

Report what the open commit collects: realize the carry through an Engine entry point that returns the commitment effects, as `reserveCloseOrderExecutionBounty` does for closes, and apply `_includeCommitment` to opens as well (`CfdOrderPolicyEvaluator.sol:65-67`).

## References

- `src/router/OrderReservationAccounting.sol:108-146`; `src/MarginClearinghouse.sol:620-655`, `:818-824`, `:1336-1356`; `src/CfdEngine.sol:535-550`, `:592-625`, `:1407-1449`
- `src/OrderLifecycleBook.sol:327-361`; `src/CfdOrderPolicyEvaluator.sol:40-68`; `src/CfdOrderPolicyEvaluatorBase.sol:325-366`, `:474-540`
- `src/router/OrderLiquidationBatchLogic.sol:236-268`, `:302-312`, `:448-454`; `src/OrderRouterAdmin.sol:376-388`; `src/OrderV3Types.sol:131-151`
- `SECURITY.md:363-384`, `:754-765`; `ACCOUNTING_SPEC.md:715-724`; `INTERNAL_ARCHITECTURE_MAP.md:93-97`; `test/perps/spec/trader/PositionProtection.t.sol:1969-2008`

## Proof of Concept

`R6_02_OpenCommitCarryBound.t.sol` (full source below) goes in `packages/perps/test/perps/poc/`. From the repository root at the commit in scope:

```bash
forge test --root packages/perps --match-path test/perps/poc/R6_02_OpenCommitCarryBound.t.sol -vv
```

The test extends the suite's `BasePerpTest` (unmodified in-scope contracts; only Pyth and USDG are mocked) with the release risk parameters, and sets the release minimum open notional through the 72 h owner timelock. Orders go through the V3 `commitOrder(OrderRequest)` and `executeOrder` that `LegacyOrderRouterHarness` inherits unchanged.

1. A 1,000-lot LONG is opened and held for 28 days.
2. A dry run reads a 20-lot increase's own execution gross debit and action charge, then rolls back.
3. The increase is committed with exactly those limits: the commit realizes the carry (`CarryRealized`), the commitment records 0, and the order executes. Each limit one atom lower fails at execution.
4. From the same state, a 20-lot partial close pinned to the same limits reverts at commit with `CommitmentBoundExceeded`, citing that carry.

Selected lines of the `-vv` output:

```text
[PASS] test_R6_02_Control_SameSizeCloseCountsTheSameCarryAgainstItsBounds() (gas: 4282710)
     commit reverts CommitmentBoundExceeded(GrossAccountDebit | ActionCharge), actual 31134060
[PASS] test_R6_02_OpenIncreaseCommitCarryBypassesPinnedBounds() (gas: 8901200)
     pinned maxGrossAccountDebitAssets / maxActionChargeAssets 3015817 2815817
     commit tx: carry realized from the trader (CarryRealized) 31134060
     commit record carryCollectedAssets 0
     account debit for the order, commit + execution 34149877
     excess over the pinned gross-debit limit 31134060
     either limit - 1: Failed ConstraintViolation on that limit
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 42.36ms (30.36ms CPU time)
```

### `R6_02_OpenCommitCarryBound.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

/// @dev R6-02 PoC: the carry an increase's commit collects is outside the order's pinned gross-debit and
///      action-charge limits, while a close commit counts the same carry against them. Release risk parameters and
///      minimum open notional (perps/DEPLOYMENT.md:130-148); the only mocks are Pyth and USDG.
contract R6_02_OpenCommitCarryBoundTest is BasePerpTest {

    address internal constant TRADER = address(0x6202);
    address internal constant KEEPER = address(0x6203);
    /// @dev One lot is 100 tokens, i.e. 100 USDG of notional at a 1.00 mark.
    uint256 internal constant LOT = 1e20;
    uint256 internal constant POSITION_LOTS = 1000;
    uint256 internal constant POSITION_MARGIN = 10_000e6;
    /// @dev Size of both the increase and the control's partial close.
    uint256 internal constant ORDER_LOTS = 20;
    uint256 internal constant INCREASE_MARGIN = 200e6;
    /// @dev Four weeks from the Thursday setup clock stays in live trading time.
    uint256 internal constant HOLD = 28 days;
    uint256 internal constant MARK = 1e8;

    /// @dev Release risk parameters (perps/DEPLOYMENT.md:130-142).
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

    /// @dev The trader opens a LONG and holds it for four weeks, so the position has accrued carry.
    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1000e6; // perps/DEPLOYMENT.md:148
        _setRouterConfig(config);
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");

        _refreshMark();
        _fundTrader(TRADER, 50_000e6);
        uint64 openId = _commit(_request(false, POSITION_LOTS, POSITION_MARGIN));
        assertEq(uint8(_execute(openId).status), uint8(OrderV3Types.LifecycleStatus.Executed), "position opens");

        vm.warp(vm.getBlockTimestamp() + HOLD);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark();
        assertFalse(engine.isOracleFrozen() || engine.isFadWindow(), "live market");
    }

    /// @dev The increase executes with limits equal to its own execution assessment, yet the account also loses the
    ///      carry its commit collected. Either limit one unit lower fails at execution, so both limits bind.
    function test_R6_02_OpenIncreaseCommitCarryBypassesPinnedBounds() public {
        (uint256 carry, uint256 maxGross, uint256 maxAction) = _dryRunIncrease();
        uint256 snapshot = vm.snapshotState();

        uint256 balanceBefore = clearinghouse.balanceAssets(TRADER);
        (uint64 orderId, uint256 commitCarry) = _commitCollectingCarry(_bounded(false, maxGross, maxAction));
        OrderV3Types.OrderReceipt memory receipt = _execute(orderId);
        uint256 debited = balanceBefore - clearinghouse.balanceAssets(TRADER);

        assertEq(uint8(receipt.status), uint8(OrderV3Types.LifecycleStatus.Executed), "increase executes");
        assertGt(commitCarry, 0, "the commit collects accrued carry");
        assertEq(commitCarry, carry, "replay is deterministic");
        assertEq(receipt.commitment.carryCollectedAssets, 0, "the open commitment records zero carry");
        assertEq(receipt.economics.grossAccountDebitAssets, maxGross, "execution gross debit sits at the limit");
        assertEq(receipt.economics.actionChargeAssessedAssets, maxAction, "execution action charge sits at the limit");
        assertEq(debited, commitCarry + maxGross, "account debit = commit carry + execution gross debit");

        console2.log("== Increase after four weeks of accrued carry (USDG atoms) ==");
        console2.log("   pinned maxGrossAccountDebitAssets / maxActionChargeAssets", maxGross, maxAction);
        console2.log("   commit tx: carry realized from the trader (CarryRealized)", commitCarry);
        console2.log("   commit record carryCollectedAssets", receipt.commitment.carryCollectedAssets);
        console2.log("   account debit for the order, commit + execution", debited);
        console2.log("   excess over the pinned gross-debit limit", debited - maxGross);

        assertTrue(vm.revertToState(snapshot), "restore before the gross-debit replay");
        (uint64 grossId,) = _commitCollectingCarry(_bounded(false, maxGross - 1, type(uint256).max));
        _assertConstraintFailure(_execute(grossId), OrderV3Types.ConstraintKind.GrossAccountDebit, maxGross);

        assertTrue(vm.revertToState(snapshot), "restore before the action-charge replay");
        (uint64 actionId,) = _commitCollectingCarry(_bounded(false, type(uint256).max, maxAction - 1));
        _assertConstraintFailure(_execute(actionId), OrderV3Types.ConstraintKind.ActionCharge, maxAction);
        console2.log("   either limit - 1: Failed ConstraintViolation on that limit");
    }

    /// @dev Control from the same state: a same-size partial close pinned to the increase's exact limits is rejected
    ///      at commit, because its commitment counts the same accrued carry against both limits.
    function test_R6_02_Control_SameSizeCloseCountsTheSameCarryAgainstItsBounds() public {
        (uint256 carry, uint256 maxGross, uint256 maxAction) = _dryRunIncrease();
        _expectCommitmentBoundExceeded(
            _bounded(true, maxGross, type(uint256).max), OrderV3Types.ConstraintKind.GrossAccountDebit, carry, maxGross
        );
        _expectCommitmentBoundExceeded(
            _bounded(true, type(uint256).max, maxAction), OrderV3Types.ConstraintKind.ActionCharge, carry, maxAction
        );

        console2.log("== Control: same-size partial close pinned to the increase's limits (USDG atoms) ==");
        console2.log("   commit reverts CommitmentBoundExceeded(GrossAccountDebit | ActionCharge), actual", carry);
    }

    /// @return carry Carry the increase's commit collects.
    /// @return gross The increase's assessed gross account debit with every limit open.
    /// @return action The increase's assessed action charge with every limit open.
    function _dryRunIncrease() internal returns (uint256 carry, uint256 gross, uint256 action) {
        uint256 snapshot = vm.snapshotState();
        uint64 orderId;
        (orderId, carry) = _commitCollectingCarry(_bounded(false, type(uint256).max, type(uint256).max));
        OrderV3Types.OrderReceipt memory receipt = _execute(orderId);
        assertEq(uint8(receipt.status), uint8(OrderV3Types.LifecycleStatus.Executed), "unbounded increase executes");
        gross = receipt.economics.grossAccountDebitAssets;
        action = receipt.economics.actionChargeAssessedAssets;
        assertTrue(vm.revertToState(snapshot), "restore after the dry run");
    }

    /// @return orderId Committed order id.
    /// @return carry Settlement the commit took from TRADER, cross-checked against the commit's `CarryRealized`.
    function _commitCollectingCarry(
        OrderV3Types.OrderRequest memory request
    ) internal returns (uint64 orderId, uint256 carry) {
        uint256 balanceBefore = clearinghouse.balanceAssets(TRADER);
        vm.recordLogs();
        orderId = _commit(request);
        carry = balanceBefore - clearinghouse.balanceAssets(TRADER);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        uint256 realized;
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter == address(engine) && logs[i].topics.length == 2
                    && logs[i].topics[0] == ICfdEngineTypes.CarryRealized.selector
                    && address(uint160(uint256(logs[i].topics[1]))) == TRADER
            ) {
                (uint256 realizedCarryAssets,,,) = abi.decode(logs[i].data, (uint256, uint256, uint256, uint256));
                realized += realizedCarryAssets;
            }
        }
        assertEq(realized, carry, "commit balance drop is exactly the realized carry");
    }

    function _commit(
        OrderV3Types.OrderRequest memory request
    ) internal returns (uint64 orderId) {
        vm.prank(TRADER);
        orderId = router.commitOrder(request);
    }

    function _expectCommitmentBoundExceeded(
        OrderV3Types.OrderRequest memory request,
        OrderV3Types.ConstraintKind kind,
        uint256 actual,
        uint256 limit
    ) internal {
        vm.prank(TRADER);
        vm.expectRevert(
            abi.encodeWithSelector(
                IOrderLifecycleBook.OrderLifecycleBook__CommitmentBoundExceeded.selector, kind, actual, limit
            )
        );
        router.commitOrder(request);
    }

    /// @return receipt The order's terminal receipt from the Book's `OrderFinalized` event.
    function _execute(
        uint64 orderId
    ) internal returns (OrderV3Types.OrderReceipt memory receipt) {
        bytes[] memory data = _mockPythUpdateData(MARK);
        vm.recordLogs();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertTrue(result.status != OrderV3Types.LifecycleStatus.Pending, "order reaches a terminal state");
        Vm.Log[] memory logs = vm.getRecordedLogs();
        address book = address(router.lifecycleBook());
        uint256 matches;
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter == book && logs[i].topics.length == 4
                    && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector
                    && uint256(logs[i].topics[1]) == orderId
            ) {
                (,,, receipt) = abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                ++matches;
            }
        }
        assertEq(matches, 1, "exactly one terminal receipt");
    }

    /// @dev The evaluator's failure evidence names the tightened limit and the execution-only assessed value.
    function _assertConstraintFailure(
        OrderV3Types.OrderReceipt memory receipt,
        OrderV3Types.ConstraintKind kind,
        uint256 assessed
    ) internal view {
        assertEq(uint8(receipt.status), uint8(OrderV3Types.LifecycleStatus.Failed), "limit - 1 fails");
        assertEq(uint8(receipt.reason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "on a constraint");
        assertEq(uint8(receipt.failure.constraint), uint8(kind), "on the tightened limit");
        assertEq(receipt.failure.actual, assessed, "evaluator counts only the execution's own amount");
        assertEq(receipt.failure.limit, assessed - 1, "against the pinned limit");
    }

    /// @dev The 20-lot increase (with `INCREASE_MARGIN`) or the 20-lot partial close, with the two limits under test.
    function _bounded(
        bool isClose,
        uint256 maxGross,
        uint256 maxAction
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request = _request(isClose, ORDER_LOTS, isClose ? 0 : INCREASE_MARGIN);
        request.bounds.maxGrossAccountDebitAssets = maxGross;
        request.bounds.maxActionChargeAssets = maxAction;
    }

    /// @dev Fresh LONG V3 request with every financial limit open; tests tighten the limits they exercise.
    function _request(
        bool isClose,
        uint256 lots,
        uint256 margin
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256(abi.encode("r6-02", router.nextCommitId(), vm.getBlockTimestamp()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = lots * LOT;
        request.marginDelta = margin;
        request.isClose = isClose;
        request.targetPrice = isClose ? CAP_PRICE : 1;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7, // Live | Fad | Frozen
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

    function _refreshMark() internal {
        bytes[] memory data = _mockPythUpdateData(MARK);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

}
```
