# \#99879 [SC-Low] Unreported commit-time carry on open/increase orders bypasses the trader's maxActionChargeAssets and maxGrossAccountDebitAssets limits

**Submitted on Nov 15th 2026 at 16:14:37 UTC by @athena for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99879
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Every V3 order carries two financial limits signed by the trader, `maxActionChargeAssets` and `maxGrossAccountDebitAssets`, which the protocol defines as inclusive caps where "zero is a real zero allowance". On open/increase orders these limits do not cover the accrued carry that the commit transaction itself collects. Locking the opening bounty realizes the carry, but the router reports `carryCollectedAssets = 0` to `OrderLifecycleBook`, and at execution the carry is no longer pending, so neither check counts it. In production, a trader who caps an increase at 50 USDG of action charges is charged 104.20 USDG and the order executes; even a zero allowance does not stop 74.25 USDG from being taken. The same carry under the same zero limit makes a close commit revert.

## Vulnerability Details

**1. The open commit realizes carry but reports none.** `OrderCommitHandler._createPendingOrder` reserves the bounty and passes the returned effects to `recordCommitment`. For a non-close order the effects contain only the bounty:

```solidity
// router/OrderCommitHandler.sol:27-32
CfdEnginePlanTypes.CloseCommitment memory effects =
    _reserveExecutionBounty(account, order.sizeDelta, executionBountyAssets, order.isClose);
...
_onOrderCommitted(order, effects);   // -> lifecycleBook.recordCommitment(orderId, effects)

// router/OrderReservationAccounting.sol:120-127 (non-close branch)
effects.bountyFromFreeAssets = bounty;          // effects.carryCollectedAssets stays 0
...
clearinghouse.lockReservedSettlement(account, bounty);

// MarginClearinghouse.sol:818-823
function lockReservedSettlement(address account, uint256 amountAssets) external onlyReservedSettlementLocker {
    _checkpointCarryBeforeMarginChange(account);   // realizes accrued carry -> HousePool
    _lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
}
```

The checkpoint reaches `CfdEngine._realizeCarryFromSettlement`, which takes the carry from position margin and then free settlement and pays it to `HousePool`. `reserveCommittedOrderMargin` (`MarginClearinghouse.sol:632`), later in the same transaction, also checkpoints without reporting.

**2. The commitment check never sees it.** `OrderLifecycleBook.sol:335-347` checks `effects.carryCollectedAssets + bounty <= maxGrossAccountDebitAssets` and `effects.carryCollectedAssets <= maxActionChargeAssets`. For opens the value is 0, so both pass. The admission check (`router/OrderLiquidationBatchLogic.sol:246-250`) compares only the bounty.

**3. The execution check does not see it either.** The evaluator adds pending carry to an open's gross debit and action charge (`CfdOrderPolicyEvaluatorBase.sol:342-347`, `:562-571`). The carry is no longer pending, and the commitment add-back `_includeCommitment` is applied only to closes (`CfdOrderPolicyEvaluator.sol:65-67`). The receipt reports `commitment.carryCollectedAssets = 0`.

**Why this is not intended behavior**
- If the commit had not realized it, the execution check above would count this carry and the 50 USDG order would fail its own limit. Carry escapes only because the commit collects it one step earlier without recording it.
- The close path does the missing steps: it returns carry in `effects.carryCollectedAssets`, `recordCommitment` bounds it, and `_includeCommitment` adds it back at execution.
- The team removed carry checkpoints from other reservation steps for this reason. `SECURITY.md:535-539`: cancel-and-close releases the trigger bounty "without checkpointing carry ... All commitment carry is then collected and recorded by the canonical close path, including its gross-debit and action-charge bounds", tested by `test_CancelAndCommitClose_AccruedCarryCannotBypassGrossDebitBound` / `...ActionChargeBound`. `SECURITY.md:384`: "Opening-bounty release does not checkpoint carry". The opening-bounty lock still does.
- `ACCOUNTING_SPEC.md:723-724`: "Receipts and financial bounds include the direct collection once". Here it is included zero times. No document exempts open-commit carry from the limits.

**Known issues.** In the Cecuro report (`audits/plether-ai-audit-report-2026-10-26.pdf`), LOW-34 ("Integrators can overstate close carry because the evaluator reports pending carry after collection", acknowledged and addressed by documentation) is about how the carry field of a close assessment is interpreted. It does not cover carry collected by an open/increase commit that is never added to the assessment at all. LOW-53 (previews overstate committable margin) and MED-22 (carry rounding when checkpointing early) are unrelated. The same goes for the AI scan item "Frequent position checkpoints discard fractional carry" (`audits/scan-plether-fi-plether-core-d4541b49.md`) and the "LP-Capital Carry" section of `ACCOUNTING_SPEC.md`. None of them covers limits on open/increase commits.

**Recommendation.** Either return the commit-time carry in `effects.carryCollectedAssets` (so `recordCommitment` bounds it) and add it back in the open execution assessment as `_includeCommitment` does for closes, or lock the opening bounty and reserve committed margin without checkpointing carry, as already done for bounty release and cancel-and-close.

## Impact Details

Release parameters; a 100,000 USDG long with 60 days of accrued carry (74.25 USDG) commits a 20,000 USDG increase:

| Trader's limit on the increase | Result |
|---|---|
| `maxActionChargeAssets = 50 USDG` | Executed. 74.25 carry at commit + 29.95 at execution = **104.20 USDG** charged. Receipt shows 0 commitment carry. |
| `maxGrossAccountDebitAssets = 31 USDG` | Executed. Execution assesses 30.15 USDG; total debit for the order is **104.40 USDG**. |
| `maxActionChargeAssets = 0`, `maxGrossAccountDebitAssets = 0.2 USDG` (bounty) | 74.25 USDG taken at commit. Order then fails on `GrossAccountDebit` and the trader's 0.2 USDG bounty goes to the keeper. A correctly bounded commit would have reverted. |
| Same carry, close commit, `maxActionChargeAssets = 0` | Reverts `CommitmentBoundExceeded(ActionCharge, 74.25 USDG, 0)`. |

The limits are the trader's protection against an order costing more than they signed for (`SECURITY.md:368`, "Bounded authority"). On increase orders the protocol moves funds out of the trader's account above that limit and executes orders their own limits should reject. The carry itself is owed, so the protocol loses no value, which is why I classify this as Low: "Contract fails to deliver promised returns, but doesn't lose value".

It affects every successful increase commit on a position with accrued carry under release parameters (`baseCarryBps = 500`). No attacker, privileged role or special timing is needed.


## References

All paths are relative to `packages/perps/` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

- `src/router/OrderCommitHandler.sol:21-35`
- `src/router/OrderReservationAccounting.sol:108-128`
- `src/MarginClearinghouse.sol:632`, `:818-823`
- `src/OrderLifecycleBook.sol:327-347`
- `src/router/OrderLiquidationBatchLogic.sol:246-250`
- `src/CfdOrderPolicyEvaluatorBase.sol:342-347`, `:562-571`
- `src/CfdOrderPolicyEvaluator.sol:65-67`
- `src/OrderV3Types.sol:131-136`
- `SECURITY.md:368`, `:384`, `:535-539`
- `ACCOUNTING_SPEC.md:723-724`
- `test/perps/spec/trader/PositionProtection.t.sol:1969-2007`



## Proof of Concept

The test runs in the repository's Foundry suite at the in-scope commit (`BasePerpTest` fixture, about 1,002,000 USDG pool depth), with the release risk parameters from `DEPLOYMENT.md`. All actions go through public entrypoints:

- `MarginClearinghouse.deposit`;
- the production V3 `OrderRouter.commitOrder(OrderRequest)`;
- `OrderRouter.executeOrder`, called by an arbitrary keeper;
- `OrderRouter.updateMarkPrice`.

Only Pyth price updates are mocked.

Steps:
1. A trader opens a 100,000 USDG LONG with 5,000 USDG margin. A keeper executes it.
2. 60 days pass and a fresh mark is installed. The position has accrued carry C = 74.248136 USDG.
3. **Control:** a close commit with `maxActionChargeAssets = 0` reverts with `CommitmentBoundExceeded(ActionCharge, C, 0)`.
4. An increase commit with `maxActionChargeAssets = 50 USDG` is accepted and debits exactly C, which goes to HousePool. The order executes. The total charge is 104.20 USDG, and the receipt reports 0 commitment carry.
5. An increase commit with `maxGrossAccountDebitAssets = 31 USDG` executes; the order's aggregate debit is 104.40 USDG.
6. An increase commit with `maxActionChargeAssets = 0` and `maxGrossAccountDebitAssets = bounty` is accepted and debits C. The order then fails at execution with `ConstraintViolation` on `GrossAccountDebit`, and the keeper receives the trader's bounty.

Save as `packages/perps/test/perps/poc/IncreaseCommitCarryBypassesBounds.t.sol` and run from the repository root:

```bash
forge test --root packages/perps --match-path "test/perps/poc/IncreaseCommitCarryBypassesBounds.t.sol" -vv
```

Output:

```
Ran 4 tests for test/perps/poc/IncreaseCommitCarryBypassesBounds.t.sol:IncreaseCommitCarryBypassesBoundsTest
[PASS] test_IncreaseCommitCollectsCarryAboveActionChargeCap()
Logs:
  maxActionChargeAssets (USDG atoms)          : 50000000
  carry collected at commit (USDG atoms)      : 74248136
  action charge collected at execution        : 29953382
  total action charges for this order         : 104201518
  receipt.commitment.carryCollectedAssets     : 0
[PASS] test_IncreaseCommitCollectsCarryAboveGrossDebitCap()
Logs:
  maxGrossAccountDebitAssets (USDG atoms)     : 31000000
  carry collected at commit (USDG atoms)      : 74248136
  bounty + execution charge                   : 30153382
  aggregate gross debit for this order        : 104401518
[PASS] test_IncreaseCommitIgnoresZeroAllowance()
Logs:
  maxActionChargeAssets                       : 0
  maxGrossAccountDebitAssets (= bounty)       : 200000
  carry collected at commit (USDG atoms)      : 74248136
  gross debit at commit (carry + bounty)      : 74448136
  execution status (3 = Failed)               : 3
  failure constraint (3 = GrossAccountDebit)  : 3
  bounty credited to the keeper (USDG atoms)  : 200000
[PASS] test_control_CloseCommitRejectsCarryAboveActionChargeCap()
Logs:
  accrued carry C collected by a close commit (USDG atoms): 74248136
Suite result: ok. 4 passed; 0 failed; 0 skipped
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {console2} from "forge-std/console2.sol";
import {Vm} from "forge-std/Vm.sol";

/// @title Open/increase commits collect accrued carry outside the order's financial bounds
/// @notice Setup (release risk parameters, BasePerpTest fixture with ~1,002,000 USDG pool depth):
///         1. A trader opens a 100,000 USDG LONG with 5,000 USDG margin.
///         2. 60 days pass; the position accrues carry C (~74.25 USDG). A fresh mark is installed.
///         Then the trader commits a new order carrying `maxActionChargeAssets` / `maxGrossAccountDebitAssets` limits.
///
///         Expected (OrderV3Types.sol:131-136, SECURITY.md "Bounded authority"): carry collected because of the commit
///         is bounded by the order's limits, as it is for close commits (OrderLifecycleBook.recordCommitment).
///         Actual: for an open/increase, lockReservedSettlement() realizes C at commit while the router reports
///         carryCollectedAssets = 0, so neither the commitment check nor the execution-time check sees C.
///
///         Only public entrypoints are used. Only Pyth price updates are mocked.
///         Run: forge test --root packages/perps --match-path "test/perps/poc/IncreaseCommitCarryBypassesBounds.t.sol" -vv
contract IncreaseCommitCarryBypassesBoundsTest is BasePerpTest {

    address internal constant TRADER = address(0xC0A1);
    address internal constant KEEPER = address(0xC0A2);
    uint256 internal constant PRICE = 1e8;

    uint256 internal constant POSITION_SIZE = 100_000e18; // 100,000 USDG notional at 1.00
    uint256 internal constant POSITION_MARGIN = 5_000e6;
    uint256 internal constant INCREASE_SIZE = 20_000e18; // 20,000 USDG notional
    uint256 internal constant INCREASE_MARGIN = 1_000e6;

    /// @dev Release risk parameters (packages/perps/DEPLOYMENT.md).
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
        _fundTrader(TRADER, 20_000e6);

        uint64 openId = _commit(_request(false, POSITION_SIZE, POSITION_MARGIN, type(uint256).max, type(uint256).max));
        (OrderV3Types.ExecutionResult memory res,) = _execute(openId);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "setup: position opened");

        vm.warp(block.timestamp + 60 days); // carry accrues on the open position
        router.updateMarkPrice(_mockPythUpdateData(PRICE)); // fresh mark so new commits are admitted
    }

    /// @notice Control: on a CLOSE commit the protocol bounds the same accrued carry.
    ///         A close with maxActionChargeAssets = 0 reverts with CommitmentBoundExceeded(ActionCharge, C, 0).
    function test_control_CloseCommitRejectsCarryAboveActionChargeCap() public {
        uint256 carry = _carryCollectedByCloseCommit();
        console2.log("accrued carry C collected by a close commit (USDG atoms):", carry);
        assertGt(carry, 0);

        OrderV3Types.OrderRequest memory closeReq = _request(true, POSITION_SIZE / 2, 0, type(uint256).max, 0);
        vm.prank(TRADER);
        vm.expectRevert(
            abi.encodeWithSelector(
                IOrderLifecycleBook.OrderLifecycleBook__CommitmentBoundExceeded.selector,
                OrderV3Types.ConstraintKind.ActionCharge,
                carry,
                uint256(0)
            )
        );
        router.commitOrder(closeReq);
    }

    /// @notice Main case: the trader caps this increase's action charges at 50 USDG (enough for its own fee + VPI,
    ///         not for C). The commit still collects C, the order executes, and the trader pays more than the cap.
    ///         The authenticated receipt reports zero commitment carry.
    function test_IncreaseCommitCollectsCarryAboveActionChargeCap() public {
        uint256 carryOnClosePath = _carryCollectedByCloseCommit();
        uint256 cap = 50e6;

        uint256 balanceBefore = clearinghouse.balanceAssets(TRADER);
        uint256 poolBefore = settlementToken.balanceOf(address(pool));

        uint64 id = _commit(_request(false, INCREASE_SIZE, INCREASE_MARGIN, type(uint256).max, cap));

        uint256 carryAtCommit = balanceBefore - clearinghouse.balanceAssets(TRADER);
        assertEq(carryAtCommit, carryOnClosePath, "same carry C that the close path bounds");
        assertEq(settlementToken.balanceOf(address(pool)) - poolBefore, carryAtCommit, "C paid to HousePool");
        assertGt(carryAtCommit, cap, "commit alone exceeds maxActionChargeAssets");

        (OrderV3Types.ExecutionResult memory res, OrderV3Types.OrderReceipt memory rc) = _execute(id);
        uint256 totalCharged = carryAtCommit + rc.economics.actionChargeCollectedAssets;

        console2.log("maxActionChargeAssets (USDG atoms)          :", cap);
        console2.log("carry collected at commit (USDG atoms)      :", carryAtCommit);
        console2.log("action charge collected at execution        :", rc.economics.actionChargeCollectedAssets);
        console2.log("total action charges for this order         :", totalCharged);
        console2.log("receipt.commitment.carryCollectedAssets     :", rc.commitment.carryCollectedAssets);

        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
        assertGt(totalCharged, cap, "trader charged above its maxActionChargeAssets");
        assertEq(rc.commitment.carryCollectedAssets, 0, "receipt omits the carry collected at commit");
    }

    /// @notice Zero allowance: maxActionChargeAssets = 0 and maxGrossAccountDebitAssets = bounty.
    ///         The close path would reject this commit; the increase commit is accepted and debits C.
    ///         The order then fails at execution on its own bounds, and the trader also forfeits the bounty.
    function test_IncreaseCommitIgnoresZeroAllowance() public {
        uint256 bounty = _openBounty();
        uint256 balanceBefore = clearinghouse.balanceAssets(TRADER);
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);

        uint64 id = _commit(_request(false, INCREASE_SIZE, INCREASE_MARGIN, bounty, 0));

        uint256 carryAtCommit = balanceBefore - clearinghouse.balanceAssets(TRADER);
        console2.log("maxActionChargeAssets                       :", uint256(0));
        console2.log("maxGrossAccountDebitAssets (= bounty)       :", bounty);
        console2.log("carry collected at commit (USDG atoms)      :", carryAtCommit);
        console2.log("gross debit at commit (carry + bounty)      :", carryAtCommit + bounty);
        assertGt(carryAtCommit, 0, "zero action-charge allowance exceeded");
        assertGt(carryAtCommit + bounty, bounty, "gross-debit allowance exceeded");

        (OrderV3Types.ExecutionResult memory res, OrderV3Types.OrderReceipt memory rc) = _execute(id);
        uint256 keeperGain = clearinghouse.balanceAssets(KEEPER) - keeperBefore;
        console2.log("execution status (3 = Failed)               :", uint8(res.status));
        console2.log("failure constraint (3 = GrossAccountDebit)  :", uint8(rc.failure.constraint));
        console2.log("bounty credited to the keeper (USDG atoms)  :", keeperGain);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Failed), "order fails on its own bounds");
        assertEq(uint8(rc.reason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "failure is a bound");
        assertEq(
            uint8(rc.failure.constraint), uint8(OrderV3Types.ConstraintKind.GrossAccountDebit), "gross-debit bound"
        );
        assertEq(keeperGain, bounty, "trader forfeits the bounty for an order its bounds should have rejected");
    }

    /// @notice Gross-debit cap: maxGrossAccountDebitAssets = 31 USDG, action charge unlimited.
    ///         Execution assesses only bounty + trade cost (~30.15 USDG), so the order executes, while the
    ///         order's aggregate account debit (commit carry + bounty + execution charge) is ~104.40 USDG.
    function test_IncreaseCommitCollectsCarryAboveGrossDebitCap() public {
        uint256 cap = 31e6;
        uint256 bounty = _openBounty();
        uint256 balanceBefore = clearinghouse.balanceAssets(TRADER);

        uint64 id = _commit(_request(false, INCREASE_SIZE, INCREASE_MARGIN, cap, type(uint256).max));
        uint256 carryAtCommit = balanceBefore - clearinghouse.balanceAssets(TRADER);

        (OrderV3Types.ExecutionResult memory res, OrderV3Types.OrderReceipt memory rc) = _execute(id);
        uint256 grossDebit = carryAtCommit + bounty + rc.economics.actionChargeCollectedAssets;

        console2.log("maxGrossAccountDebitAssets (USDG atoms)     :", cap);
        console2.log("carry collected at commit (USDG atoms)      :", carryAtCommit);
        console2.log("bounty + execution charge                   :", bounty + rc.economics.actionChargeCollectedAssets);
        console2.log("aggregate gross debit for this order        :", grossDebit);

        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
        assertGt(grossDebit, cap, "trader debited above its maxGrossAccountDebitAssets");
        assertEq(rc.commitment.carryCollectedAssets, 0, "receipt omits the carry collected at commit");
    }

    // helpers

    function _request(
        bool isClose,
        uint256 size,
        uint256 margin,
        uint256 maxGrossAccountDebit,
        uint256 maxActionCharge
    ) internal view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = keccak256(abi.encode("cx-r1", router.nextCommitId()));
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.isClose = isClose;
        r.targetPrice = isClose ? 2e8 : 1; // non-binding slippage limit for a LONG
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: maxGrossAccountDebit,
            maxActionChargeAssets: maxActionCharge,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _commit(
        OrderV3Types.OrderRequest memory r
    ) internal returns (uint64 id) {
        vm.prank(TRADER);
        id = router.commitOrder(r);
    }

    /// @dev Permissionless keeper execution with a post-commit Pyth tick; returns the order's terminal receipt.
    function _execute(
        uint64 id
    ) internal returns (OrderV3Types.ExecutionResult memory res, OrderV3Types.OrderReceipt memory receipt) {
        bytes[] memory data = _mockPythUpdateData(PRICE);
        vm.recordLogs();
        vm.prank(KEEPER);
        res = router.executeOrder(id, data);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        uint256 found;
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter == address(router.lifecycleBook()) && logs[i].topics.length == 4
                    && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector
                    && uint256(logs[i].topics[1]) == id
            ) {
                (,,, receipt) = abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                ++found;
            }
        }
        assertEq(found, 1, "one terminal receipt");
    }

    /// @dev Measures C on the close path (which bounds it), then rolls the state back.
    function _carryCollectedByCloseCommit() internal returns (uint256 carry) {
        uint256 snap = vm.snapshotState();
        uint256 balanceBefore = clearinghouse.balanceAssets(TRADER);
        _commit(_request(true, POSITION_SIZE / 2, 0, type(uint256).max, type(uint256).max));
        carry = balanceBefore - clearinghouse.balanceAssets(TRADER);
        vm.revertToStateAndDelete(snap);
    }

    /// @dev Reads the opening bounty the router reserves for INCREASE_SIZE, then rolls the state back.
    function _openBounty() internal returns (uint256 bounty) {
        uint256 snap = vm.snapshotState();
        uint64 id = _commit(_request(false, INCREASE_SIZE, INCREASE_MARGIN, type(uint256).max, type(uint256).max));
        bounty = clearinghouse.getBountyReservation(IMarginClearinghouse.BountyKind.Order, id).amountAssets;
        vm.revertToStateAndDelete(snap);
    }

}
```

