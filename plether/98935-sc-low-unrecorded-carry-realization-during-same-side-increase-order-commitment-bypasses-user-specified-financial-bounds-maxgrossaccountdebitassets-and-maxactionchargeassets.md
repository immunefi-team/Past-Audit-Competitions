# \#98935 [SC-Low] Unrecorded Carry Realization During Same Side Increase Order Commitment Bypasses User Specified Financial Bounds (`maxGrossAccountDebitAssets` and `maxActionChargeAssets`)

**Submitted on Nov 11th 2026 at 02:14:34 UTC by @n0fr33w1f14u for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98935
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

# Description

## Brief/Intro

When a trader with an existing open position commits a same-side increase order via OrderRouter.commitOrder(), reserving the opening execution bounty triggers MarginClearinghouse.lockReservedSettlement(), which immediately checkpoints and collects all accrued position carry from the trader's clearinghouse account and transfers it to HousePool. However, OrderReservationAccounting._reserveExecutionBounty() only populates effects.bountyFromFreeAssets and leaves effects.carryCollectedAssets, effects.settlementBeforeAssets, and effects.settlementAfterAssets as zero. Consequently, OrderLifecycleBook.recordCommitment() fails to check the collected carry against the order's maxGrossAccountDebitAssets and maxActionChargeAssets bounds at commit time, and CfdOrderPolicyEvaluator.assessCommittedOrder() omits that already-paid carry at execution time. In production, a trader's same-side increase order can debit and transfer significantly more USDG than authorized by the order's explicit financial bounds while recording incomplete commitment and receipt accounting.

## Vulnerability Details

In Plether V3, every delayed order submission includes an `OrderV3Types.ExecutionBounds` struct that specifies hard financial ceilings for the order, including `maxGrossAccountDebitAssets` and `maxActionChargeAssets`. For close orders, carry collected during order commitment is tracked in `CfdEnginePlanTypes.CloseCommitment`, validated against `maxGrossAccountDebitAssets` and `maxActionChargeAssets` in `OrderLifecycleBook.recordCommitment()`, and added back into the execution-time policy assessment via `_includeCommitment()`. For same-side increase orders (`isClose == false` when the account already holds a live position), this accounting is bypassed across five steps:

### Step 1: Commit-time preflight checks do not enforce action-charge or gross-debit bounds against pending carry
When `OrderRouter.commitOrder()` delegates to `OrderLiquidationBatchLogic._commitOrder()`, `_validatedOpenBounty()` calls `engineLensForCommit().previewOpenRevertCode()`. `CfdEnginePlanLib.planOpen()` projects pending carry collection for margin and solvency checks. The preflight in `_commitOrder()` checks the bounty against the bounty and gross-debit ceilings (`packages/perps/src/router/OrderLiquidationBatchLogic.sol:233-250`):

```solidity
if (request.isClose) {
    _validateCloseAdmission(host, account, request.side, request.sizeDelta);
} else {
    executionBountyAssets =
        _validatedOpenBounty(host, account, request.side, request.sizeDelta, request.marginDelta);
}
if (executionBountyAssets > request.bounds.maxExecutionBountyAssets) {
    revert IOrderLifecycleBook.OrderLifecycleBook__ExecutionBountyAboveBound(
        executionBountyAssets, request.bounds.maxExecutionBountyAssets
    );
}
if (executionBountyAssets > request.bounds.maxGrossAccountDebitAssets) {
    revert OrderRouter__ExecutionBountyAboveGrossDebit(
        executionBountyAssets, request.bounds.maxGrossAccountDebitAssets
    );
}
```

Furthermore, `_validateFreshRequest()` (`packages/perps/src/router/OrderLiquidationBatchLogic.sol:305-310`) rejects any nonzero `minPostSettlementBalanceAssets` when `positionSize != 0` with `OrderRouter__SettlementFloorUnsupported`, leaving `maxGrossAccountDebitAssets` and `maxActionChargeAssets` as the only bounds protecting a trader's settlement balance during a same-side increase.

### Step 2: Reserving the opening bounty triggers real carry collection but returns zero carry in `effects`
Next, `_queueCommittedOrder()` invokes `OrderRouter.executeCommitOrderItem()`, which calls `OrderCommitHandler._createPendingOrder()` (`packages/perps/src/router/OrderCommitHandler.sol:22-36`):

```solidity
function _createPendingOrder(
    CfdTypes.Order memory order,
    uint256 executionBountyAssets
) internal {
    uint64 orderId = order.orderId;
    address account = order.account;
    CfdEnginePlanTypes.CloseCommitment memory effects =
        _reserveExecutionBounty(account, order.sizeDelta, executionBountyAssets, order.isClose);
    clearinghouse.recordBountyReservation(
        account, IMarginClearinghouse.BountyKind.Order, orderId, executionBountyAssets
    );
    _onOrderCommitted(order, effects);
    _reserveCommittedMargin(account, orderId, order.isClose, order.marginDelta);

    _recordCommittedOrder(order);
}
```

In `OrderReservationAccounting._reserveExecutionBounty()` (`packages/perps/src/router/OrderReservationAccounting.sol:120-139`), the open/increase branch (`!isClose`) only assigns `effects.bountyFromFreeAssets`:

```solidity
function _reserveExecutionBounty(
    address account,
    uint256 sizeDelta,
    uint256 bounty,
    bool isClose
) internal returns (CfdEnginePlanTypes.CloseCommitment memory effects) {
    if (isClose) {
        if (bounty != 0) {
            revert ICfdEngineTypes.CfdEngine__InvalidRiskParams();
        }
        return _reserveCloseExecutionBounty(account, sizeDelta);
    }
    effects.bountyFromFreeAssets = bounty;
    if (bounty == 0) {
        return effects;
    }
    if (clearinghouse.getAccountAssetBuckets(account).freeSettlementAssets < bounty) {
        revert OrderRouter__InsufficientFreeEquity();
    }
    clearinghouse.lockReservedSettlement(account, bounty);
}
```

All other fields of `effects` (`carryCollectedAssets`, `carryOutstandingAssets`, `settlementBeforeAssets`, `settlementAfterAssets`, and `freeSettlementAfterAssets`) remain `0`.

### Step 3: `MarginClearinghouse.lockReservedSettlement()` realizes carry and transfers USDG to `HousePool`
Because the minimum opening bounty is nonzero (`minOpenOrderExecutionBountyAssets` is 0.01 USDG by default and enforced nonzero by `OrderRouterAdmin`), `clearinghouse.lockReservedSettlement(account, bounty)` is called (`packages/perps/src/MarginClearinghouse.sol:814-820`):

```solidity
function lockReservedSettlement(
    address account,
    uint256 amountAssets
) external onlyReservedSettlementLocker {
    _checkpointCarryBeforeMarginChange(account);
    _lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
}
```

`_checkpointCarryBeforeMarginChange(account)` calls `CfdEngine.realizeCarryBeforeMarginChange(account)` (`packages/perps/src/CfdEngine.sol:535-549`), which executes `_realizeCarryFromSettlement(account, pos)` whenever `pos.lots > 0` (`packages/perps/src/CfdEngine.sol:1407-1449`):
1. `_checkpointPositionCarry()` advances the side carry indexes, calculates `carryDue`, and updates `pos.lastCarryTimestamp = uint64(block.timestamp)` and `pos.lastCarryIndex = endIndex`.
2. `clearinghouse.consumeSettlementLoss()` deducts the carry from the account's active position margin (`positionMarginAssets`) first and free settlement second, debiting `settlementBalances[account]` and transferring the USDG to `CfdEngine`.
3. `CfdEngine` transfers `realizedCarryAssets` to `HousePool` and records the revenue inflow.

(Note: `_reserveCommittedMargin()` also calls `clearinghouse.reserveCommittedOrderMargin()`, which invokes `_checkpointCarryBeforeMarginChange(account)` as well.)

### Step 4: `OrderLifecycleBook.recordCommitment()` passes without checking the collected carry
`_onOrderCommitted(order, effects)` calls `OrderLifecycleBook.recordCommitment()` (`packages/perps/src/OrderLifecycleBook.sol:327-362`):

```solidity
uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;
if (gross > pending.bounds.maxGrossAccountDebitAssets) {
    revert OrderLifecycleBook__CommitmentBoundExceeded(
        OrderV3Types.ConstraintKind.GrossAccountDebit, gross, pending.bounds.maxGrossAccountDebitAssets
    );
}
if (effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets) {
    revert OrderLifecycleBook__CommitmentBoundExceeded(
        OrderV3Types.ConstraintKind.ActionCharge,
        effects.carryCollectedAssets,
        pending.bounds.maxActionChargeAssets
    );
}
if (
    effects.settlementBeforeAssets != effects.settlementAfterAssets + effects.carryCollectedAssets
        || effects.bountyFromFreeAssets != pending.executionBountyAssets
) {
    revert OrderLifecycleBook__InvalidCommitmentEffects();
}
```

Because `effects.carryCollectedAssets`, `effects.settlementBeforeAssets`, and `effects.settlementAfterAssets` are all `0`:
* `gross` evaluates to `0 + pending.executionBountyAssets`.
* `effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets` evaluates `0 > maxActionChargeAssets` (false).
* `effects.settlementBeforeAssets != effects.settlementAfterAssets + effects.carryCollectedAssets` evaluates `0 != 0 + 0` (false).
* The commitment succeeds and stores `carryCollectedAssets = 0` in `pending.commitment`.

### Step 5: Execution-time policy evaluation and receipt generation omit the commit-time carry
When a keeper executes the order via `executeOrder()` or `executeOrderBatch()`:
1. `CfdOrderPolicyEvaluator.assessCommittedOrder()` (`packages/perps/src/CfdOrderPolicyEvaluator.sol:22-67`) builds a snapshot from the current `CfdEngine` state. The planner's pending carry includes accrual since the commitment checkpoint and any unpaid carry. The carry paid at commitment has already cleared from that calculation.
2. `assessCommittedOrder()` only includes commitment effects for close orders (`packages/perps/src/CfdOrderPolicyEvaluator.sol:64-66`):
   ```solidity
   assessment = _assessSnapshot(context, order, pending.bounds, planner, snapshot);
   if (order.isClose) {
       _includeCommitment(assessment, pending.commitment, pending.bounds, pending.executionBountyAssets);
   }
   ```
3. `OrderRouterExecutionSidecar._executeAssessed()` (`packages/perps/src/OrderRouterExecutionSidecar.sol:493-517`) snapshots `preState` after the commitment debit. The state equality checks use that balance as their starting point, and the executed receipt's economics omit the earlier payment.

---

### Step-by-Step Numerical Walkthrough

Using the release parameters (`baseCarryBps = 500` (5%), `vpiFactor = 0.01e18`, `executionFeeBps = 4` (0.04%), `bountyBps = 10` (0.1%), `initMarginBps = 20` (0.2%), `maintMarginBps = 10` (0.1%), `fadMarginBps = 300` (3%), `openOrderExecutionBountyBps = 1` (0.01%)):

1. **Initial State at `t = 0`:**
   * `HousePool` cash (`poolAssets`): `1,000,000,000,000` (1,000,000 USDG), with 0 trader claims.
   * Trader holds an existing LONG position of `100,000e18` tokens (1,000 lots) opened at price `1.00` (`100,000,000`).
   * `entryCostAssetAtoms` = `100,000,000,000` (100,000 USDG) and `maxProfitAssets` = `100,000,000,000` (100,000 USDG).
   * Trader clearinghouse buckets:
     * PnL pledge (`positionMarginAssets`): `10,000,000,000` (10,000 USDG)
     * Liquidation reserve: `100,000,000` (100 USDG)
     * Free settlement: `5,000,000,000` (5,000 USDG)
     * Total settlement balance: `15,100,000,000` (15,100 USDG)
   * Position borrow base: `100,000,000,000 - 10,000,000,000 = 90,000,000,000` (90,000 USDG).
   * Side utilization: `(90,000,000,000 * 10,000) / 1,000,000,000,000 = 900` bps (9%).

2. **Carry Accrual Over 30 Days (`2,592,000` seconds):**
   * `indexDelta = (500 * 900 * 1e18 * 2,592,000) / (31,536,000 * 10,000 * 10,000) = 369,863,013,698,630`.
   * `carryDue = (90,000,000,000 * 369,863,013,698,630) / 1e18 = 33,287,671` atoms (`33.287671 USDG`).

3. **Same-Side Increase Commitment at `t = 2,592,000`:**
   * The trader submits `commitOrder()` for a `1,000e18`-token (10-lot) LONG increase at mark price `1.00` with `marginDelta = 100,000,000` (100 USDG) and explicit bounds:
     * `maxActionChargeAssets = 2,000,000` (2.00 USDG)
     * `maxGrossAccountDebitAssets = 2,100,000` (2.10 USDG)
   * Opening bounty is `(1,000,000,000 * 1) / 10,000 = 100,000` (0.10 USDG).
   * During `_reserveExecutionBounty()`, `lockReservedSettlement()` collects `33,287,671` atoms (`33.287671 USDG`) of carry from the trader's PnL pledge and transfers it to `HousePool`.
   * Despite `33.287671 USDG` exceeding both `maxActionChargeAssets` (2.00 USDG) and `maxGrossAccountDebitAssets` (2.10 USDG), `recordCommitment()` receives `carryCollectedAssets = 0` and succeeds.
   * Post-commitment PnL pledge is `9,966,712,329` (`9,966.712329 USDG`), new borrow base is `90,033,287,671`, and new pool cash is `1,000,033,287,671`.

4. **Illustrative execution one second later at `1.00`**
   * These figures assume a qualifying live oracle update at the stated execution price. The commitment debit above establishes the breach independently of execution.
   * Carry accrued over the 1-second interval:
     * `indexDelta = (500 * 900 * 1e18 * 1) / (31,536,000 * 10,000 * 10,000) = 142,694,063`.
     * `pendingCarryAssets = (90,033,287,671 * 142,694,063) / 1e18 = 12` atoms (`0.000012 USDG`).
   * VPI and execution fee on the 1,000 USDG increase against pool depth `1,000,033,287,683`:
     * `preCost = 49,998,335` atoms; `postCost = 51,003,302` atoms -> `vpiAssets = 1,004,967` atoms (`1.004967 USDG`).
     * `executionFeeAssets = (1,000,000,000 * 4) / 10,000 = 400,000` atoms (`0.400000 USDG`).
     * `tradeCostAssets = 1,004,967 + 400,000 = 1,404,967` atoms (`1.404967 USDG`).
   * Policy evaluator checks:
     * `actionChargeAssessedAssets = 12 + 1,404,967 = 1,404,979` (`1.404979 USDG <= 2.00 USDG` limit) -> **Passes**.
     * `grossAccountDebitAssets = 12 + 1,404,967 + 100,000 = 1,504,979` (`1.504979 USDG <= 2.10 USDG` limit) -> **Passes**.
   * Actual total debited across commitment and execution:
     * Total carry collected: `33.287671 + 0.000012 = 33.287683 USDG`
     * Total action charge collected: `33.287683 + 1.404967 = 34.692650 USDG` (exceeding the `2.00 USDG` bound by `32.692650 USDG`)
     * Total gross account debit: `34.692650 + 0.100000 = 34.792650 USDG` (exceeding the `2.10 USDG` bound by `32.692650 USDG`)

## Impact Details

* **In-Scope Impact Category:** Low — *"Contract fails to deliver promised returns, but doesn't lose value."*
* **Breakdown of Impact:**
  1. **Violation of User-Specified Financial Bounds:** Traders submitting a same-side increase order cannot rely on `maxGrossAccountDebitAssets` or `maxActionChargeAssets` (and are prohibited from setting a nonzero `minPostSettlementBalanceAssets` when holding an open position). The commitment transaction immediately debits accrued carry from the trader's collateral and transfers USDG to `HousePool` without enforcing the caller's configured limits.
  2. **Irreversible Commitment Debit on Failed/Expired Orders:** Even if the increase order later fails at execution (for example, due to slippage or a constraint violation) or expires, the carry collected during commitment has already been permanently debited and transferred in the commitment transaction despite exceeding the request's maximum debit bounds.
  3. **Inaccurate lifecycle and receipt accounting.** The reported commitment shows zero for `carryCollectedAssets`, `settlementAfterAssets`, and `settlementBeforeAssets`. The Book stores the first two values and derives `settlementBeforeAssets` from their sum when read. In the example, the executed receipt also omits the commitment carry from `economics.carryAssets`, `economics.actionChargeAssessedAssets`, `economics.actionChargeCollectedAssets`, and `economics.grossAccountDebitAssets`. [Commitment readout](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L482-L490)
  4. **Low severity.** Plether documents `maxGrossAccountDebitAssets` as an aggregate ceiling on settlement debits, consumed claims, and the reserved bounty. It documents `maxActionChargeAssets` as a ceiling on the assessed action charge including carry. The accepted commitment breaches both promises. The USDG transfer settles carry already owed, so the Low claim rests on the failure to honor the documented ceilings. [Financial limits](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L89-L105)

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../packages/perps/test/perps/BasePerpTest.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";

/// @notice PoC for "Unrecorded Carry Realization During Same-Side Increase Order Commitment".
/// @dev forge test --match-path test/poc/UnrecordedIncreaseCommitCarry.t.sol -vv
///      A same-side increase commitment realizes accrued carry (via lockReservedSettlement) and moves it to the
///      HousePool, but the commitment effects passed to OrderLifecycleBook.recordCommitment() report zero carry,
///      so maxActionChargeAssets / maxGrossAccountDebitAssets are not enforced. A close commitment with the very
///      same bounds and the very same accrued carry is rejected.
contract UnrecordedIncreaseCommitCarryPoC is BasePerpTest {

    address private constant ALICE = address(0xA11CE);
    uint256 private constant POSITION_SIZE = 100_000e18;
    uint256 private constant POSITION_MARGIN = 10_000e6;
    uint256 private constant INCREASE_SIZE = 1000e18;
    uint256 private constant INCREASE_MARGIN = 100e6;
    uint256 private constant MAX_ACTION_CHARGE = 2e6; // 2.00 USDG
    uint256 private constant MAX_GROSS_DEBIT = 2_100_000; // 2.10 USDG
    uint256 private constant EXECUTION_DELAY = 30;

    struct MockPrice {
        int64 price;
        uint64 conf;
        int32 expo;
        uint256 publishTime;
        uint256 prevPublishTime;
    }

    struct MockFeedUpdate {
        bytes32 id;
        MockPrice price;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _openAndAccrue() private returns (uint256 carry) {
        _fundTrader(ALICE, 20_000e6);
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN, 1e8);
        vm.warp(block.timestamp + 30 days);
        vm.roll(block.number + 1);
        carry = _expectedIndexedCarryAssets(ALICE);
        assertGt(carry, MAX_GROSS_DEBIT, "accrued carry exceeds both bounds");
    }

    /// @notice Control: a close commitment with the same bounds is rejected because the carry it collects is
    ///         recorded and checked against maxGrossAccountDebitAssets / maxActionChargeAssets.
    function test_Control_CloseCommitWithSameBoundsReverts() public {
        uint256 carry = _openAndAccrue();
        emit log_named_uint("accrued carry at commit", carry);

        OrderV3Types.OrderRequest memory request = _request("close", POSITION_SIZE, 0, true);
        vm.expectPartialRevert(IOrderLifecycleBook.OrderLifecycleBook__CommitmentBoundExceeded.selector);
        vm.prank(ALICE);
        router.commitOrder(request);
    }

    /// @notice Same-side increase with a live position cannot use the settlement-floor bound either.
    function test_IncreaseCannotUseSettlementFloor() public {
        _openAndAccrue();
        OrderV3Types.OrderRequest memory request = _request("floor", INCREASE_SIZE, INCREASE_MARGIN, false);
        request.bounds.minPostSettlementBalanceAssets = 1;
        vm.expectRevert(IOrderRouterErrors.OrderRouter__SettlementFloorUnsupported.selector);
        vm.prank(ALICE);
        router.commitOrder(request);
    }

    /// @notice Bug: the increase commitment debits the full accrued carry, sends it to the pool, records zero carry,
    ///         and succeeds despite bounds of 2.00 / 2.10 USDG.
    function test_IncreaseCommitDebitsCarryAboveBoundsAndRecordsZero() public {
        uint256 carry = _openAndAccrue();

        uint256 settlementBefore = clearinghouse.balanceAssets(ALICE);
        uint256 pledgeBefore = clearinghouse.pnlPledgeAssets(ALICE);
        uint256 poolBefore = pool.totalAssets();

        OrderV3Types.OrderRequest memory request = _request("increase", INCREASE_SIZE, INCREASE_MARGIN, false);
        vm.prank(ALICE);
        uint64 orderId = router.commitOrder(request);

        IOrderLifecycleBook book = IOrderLifecycleBook(address(router.lifecycleBook()));
        OrderV3Types.PendingIntent memory intent = book.pendingIntent(orderId);
        CfdEnginePlanTypes.CloseCommitment memory c = intent.commitment;

        uint256 settlementDebit = settlementBefore - clearinghouse.balanceAssets(ALICE);
        uint256 pledgeDebit = pledgeBefore - clearinghouse.pnlPledgeAssets(ALICE);
        uint256 poolGain = pool.totalAssets() - poolBefore;

        emit log_named_uint("accrued carry at commit         ", carry);
        emit log_named_uint("maxActionChargeAssets           ", intent.bounds.maxActionChargeAssets);
        emit log_named_uint("maxGrossAccountDebitAssets      ", intent.bounds.maxGrossAccountDebitAssets);
        emit log_named_uint("execution bounty reserved       ", intent.executionBountyAssets);
        emit log_named_uint("settlement balance debited      ", settlementDebit);
        emit log_named_uint("PnL pledge consumed             ", pledgeDebit);
        emit log_named_uint("HousePool gain                  ", poolGain);
        emit log_named_uint("recorded carryCollectedAssets   ", c.carryCollectedAssets);
        emit log_named_uint("recorded settlementBeforeAssets ", c.settlementBeforeAssets);
        emit log_named_uint("recorded settlementAfterAssets  ", c.settlementAfterAssets);

        // Real effect of the commit transaction.
        assertEq(settlementDebit, carry, "carry left the trader's settlement balance at commit");
        assertEq(pledgeDebit, carry, "carry consumed from the PnL pledge");
        assertEq(poolGain, carry, "carry transferred to HousePool at commit");
        assertEq(engine.unsettledCarryAssets(ALICE), 0);
        assertGt(carry, intent.bounds.maxActionChargeAssets, "debit exceeds maxActionChargeAssets");
        assertGt(carry + intent.executionBountyAssets, intent.bounds.maxGrossAccountDebitAssets, "exceeds gross");

        // Book records none of it.
        assertEq(c.carryCollectedAssets, 0, "commitment records zero carry");
        assertEq(c.settlementAfterAssets, 0);
        assertEq(c.settlementBeforeAssets, 0);
        assertEq(c.bountyFromFreeAssets, intent.executionBountyAssets);

        // Execution passes the policy check because the commit-time carry is gone from the assessment.
        uint64 commitTime = uint64(vm.getBlockTimestamp());
        vm.warp(uint256(commitTime) + EXECUTION_DELAY);
        vm.roll(block.number + 1);
        _startRecordingLogs();
        OrderV3Types.ExecutionResult memory result =
            router.executeOrder(orderId, _priceUpdate(1e8, commitTime + 1, commitTime));
        emit log_named_uint("execution status", uint8(result.status));
        emit log_named_uint("terminalReason  ", uint8(result.terminalReason));

        OrderV3Types.OrderReceipt memory receipt = _verifiedReceipt(book, orderId);
        assertEq(uint8(receipt.status), uint8(OrderV3Types.LifecycleStatus.Executed), "increase executed");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE + INCREASE_SIZE, "position increased");

        emit log_named_int("receipt carryAssets                ", receipt.economics.carryAssets);
        emit log_named_uint("receipt actionChargeAssessedAssets ", receipt.economics.actionChargeAssessedAssets);
        emit log_named_uint("receipt grossAccountDebitAssets    ", receipt.economics.grossAccountDebitAssets);
        assertLe(receipt.economics.actionChargeAssessedAssets, MAX_ACTION_CHARGE);
        assertLe(receipt.economics.grossAccountDebitAssets, MAX_GROSS_DEBIT);
        assertLt(uint256(receipt.economics.carryAssets), carry, "receipt omits commit-time carry");

        uint256 totalActionCharge = carry + receipt.economics.actionChargeAssessedAssets;
        emit log_named_uint("true total action charge (commit + exec)", totalActionCharge);
        assertGt(totalActionCharge, MAX_ACTION_CHARGE, "user bound breached overall");
    }

    function _request(
        bytes32 clientOrderId,
        uint256 sizeDelta,
        uint256 marginDelta,
        bool isClose
    ) private view returns (OrderV3Types.OrderRequest memory request) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        request.clientOrderId = clientOrderId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        request.targetPrice = isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
            executionWindowSeconds: window,
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: MAX_GROSS_DEBIT,
            maxActionChargeAssets: MAX_ACTION_CHARGE,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _priceUpdate(
        uint256 price,
        uint64 publishTime,
        uint64 prevPublishTime
    ) private pure returns (bytes[] memory updateData) {
        bytes32[] memory feedIds = _basePythFeedIds();
        MockFeedUpdate[] memory updates = new MockFeedUpdate[](feedIds.length);
        for (uint256 i; i < feedIds.length; ++i) {
            updates[i] = MockFeedUpdate({
                id: feedIds[i],
                price: MockPrice({
                    price: int64(uint64(price)),
                    conf: 0,
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: prevPublishTime
                })
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

}
```
