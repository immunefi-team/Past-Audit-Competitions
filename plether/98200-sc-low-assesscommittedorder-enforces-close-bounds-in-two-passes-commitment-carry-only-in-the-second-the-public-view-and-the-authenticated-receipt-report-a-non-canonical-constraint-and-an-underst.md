# \#98200 [SC-Low] assessCommittedOrder enforces close bounds in two passes, commitment carry only in the second: the public view and the authenticated receipt report a non-canonical constraint and an understated actual

**Submitted on Nov 5th 2026 at 22:33:53 UTC by @Lester for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98200
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
SECURITY.md ("Terminal receipt boundary") documents the lifecycle receipt as an authenticated, permanent record that carries typed failure evidence, i.e. which financial limit failed and by how much. ACCOUNTING_SPEC says collected close-commitment carry "is included exactly once in gross account debit, assessed/collected action charges ... for the user's financial bounds". `OrderV3Types.ConstraintKind` says "Ordering is canonical and is also the required evaluation order".

`CfdOrderPolicyEvaluator.assessCommittedOrder` enforces the close bounds in TWO passes:
- pass 1 runs on an assessment WITHOUT the carry already collected at commitment;
- only after it does `_includeCommitment` add that carry and run pass 2.

The order outcome is the same either way, because adding carry only tightens the bounds. But the evidence that the public view and the authenticated receipt report is wrong:
- **Normal case, one limit set from a pre-carry quote:** a GrossAccountDebit failure reports actual 4,000,128 while the order's real gross debit is 6,784,079. The evidence understates it by the full 2,783,951 commitment carry.
- **With a second violated limit:** the receipt names a later constraint (ExplicitFees) instead of the canonical first violation (ActionCharge, 37,406,126 > 37,406,125).

## Vulnerability Details
**Root cause**
- `CfdOrderPolicyEvaluator.sol:64-67` (`assessCommittedOrder`) calls `_assessSnapshot` for a close.
  - That calls `_evaluateClose`, then `_enforceBounds` (pass 1, `CfdOrderPolicyEvaluatorBase.sol:178-190`).
  - Pass 1 uses `grossAccountDebitAssets` and `actionChargeAssessedAssets` WITHOUT the carry already collected at commitment.
- Only then does `_includeCommitment` (`CfdOrderPolicyEvaluatorBase.sol:474-487`) add `effects.carryCollectedAssets` to the gross debit, the assessed and collected action charge, and the carry. It then runs `_enforceBounds` again (pass 2).
- The first revert wins. If pass-1 values already violate GrossAccountDebit (3) or ActionCharge (4), the reported `actual` excludes the commitment carry. If any later kind is violated (ExplicitFees (5), PostPositionSize (6), PostSettlementBalance (7) and so on), it is reported even though kind 3 or 4 is violated by the real, commitment-inclusive assessment.

**Spec vs code**
- SECURITY.md, Terminal receipt boundary: the Book authenticates the receipt and emits the full receipt, including typed failure evidence. The wrong constraint, actual and limit become the permanent authenticated record.
- `ACCOUNTING_SPEC.md:943`: "Collected commitment carry is included exactly once in gross account debit, assessed/collected action charges, and carry totals for the user's financial bounds." Pass 1 evaluates, and can report, values that exclude it.
- `OrderV3Types.sol:100` (`ConstraintKind`): "Ordering is canonical and is also the required evaluation order."
- `CfdOrderPolicyEvaluatorBase.sol:105` says "Bounds follow `ConstraintKind` order", and `:489` says "Bound precedence is deliberately identical to the order of `ConstraintKind` values."

**Not a duplicate**
- We reported the increase path separately as **#98198**, and this report is distinct from it.
  - #98198: carry collected at an increase commitment is never bounded. Its fix is to record and bound it.
  - Here: the close path does bound commitment carry, but in the wrong pass. The fix is to include it before a single `_enforceBounds`.
  - The root causes differ, the code paths differ, and the fixes differ.
- Cecuro LOW-34 ("Integrators can overstate close carry because the evaluator reports pending carry after collection") also touches `_includeCommitment`. It concerns the reported `carryAssets` amount, not the two-pass enforcement, and its fix does not change the bound evaluation.
- No public plether-core commit changes the bound order.

## Impact Details
**Impact (Low):** "Contract fails to deliver promised returns, but doesn't lose value".

This is a logic error in an in-scope external function and in the documented receipt guarantee. The public `CfdOrderPolicyEvaluator.assessCommittedOrder` (external; reverts `ConstraintViolation(kind, actual, limit)`) and the authenticated terminal receipt (`failure.constraint/actual/limit`) report evidence that contradicts the "included exactly once" rule and the canonical order.

Who pays: a trader, integrator or position-protection owner who re-quotes from the receipt.
- A typical automated re-quote raises the reported limit by the reported shortfall and resubmits.
  - In the normal case it raises the gross-debit limit by 1 atom against a real shortfall of 2,783,952 atoms.
  - In the two-limit case it raises ExplicitFees, while ActionCharge is the real blocker.
- Each misled round costs a full extra commit plus keeper execution, and delays the close while the position stays open.
- Ordinary and protection closes carry a zero execution bounty (SECURITY.md:374), so the per-round cost is gas and delay, not a bounty.

The order outcome itself is identical, and no funds move differently. That is why this is Low.

**Suggested fix:** in `assessCommittedOrder`, include the commitment effects before a single `_enforceBounds` call. Build the close assessment, add the commitment carry, then enforce once.

## References
- Code (frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`):
  - `packages/perps/src/CfdOrderPolicyEvaluator.sol#L64-L67`
  - `packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L105`, `#L178-L190`, `#L474-L489`
  - `packages/perps/src/OrderV3Types.sol#L100`
- Spec:
  - `packages/perps/ACCOUNTING_SPEC.md#L943`
  - `packages/perps/SECURITY.md` (Terminal receipt boundary; `#L374` zero close bounty)
- Related report by the same researcher, different root cause and fix: #98198.

## Proof of Concept

## Proof of Concept
This is a runnable Foundry test in the repository's own perps test suite, at the frozen commit, with no change to any in-scope contract. It uses the repo fixture `BasePerpTest`; its MockPyth and mock USDG are the allowed out-of-scope mocks. All actions go through public entrypoints:
- the trader calls `router.commitOrder`;
- an untrusted keeper calls `router.executeOrder` and the permissionless `router.updateMarkPrice`;
- the view is read through `policyEvaluator.assessCommittedOrder`.

The full test code is included below.

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the test code below as packages/perps/test/PocD_CloseBoundOrder.t.sol
forge test --root packages/perps --match-path test/PocD_CloseBoundOrder.t.sol -vv   # Foundry 1.5.1
```

No env or RPC is needed. Expected: `2 passed`.

**`test_D1b_GrossDebitEvidenceOmitsCommitmentCarry`** is the normal case: a single limit, set from a quote taken before the commitment carry was collected. The reported kind is right, but the evidence is understated:
- 6 h of carry give a commitment carry of 2,783,951 and a true gross debit of 6,784,079.
- `maxGrossAccountDebitAssets` = 4,000,127. The Book accepts it at commitment.
- The receipt reports GrossAccountDebit with actual 4,000,128 and limit 4,000,127, which understates the real gross debit by the full 2,783,951 commitment carry.

**`test_D1_CloseBoundFailureReportsNonCanonicalConstraint`** has the same root cause and adds a second violated limit. It shows that the receipt also breaks the canonical constraint order (USDG atoms):
1. The trader opens a 300k LONG, and 3 days of carry accrue.
2. A dry run measures a 10k partial close:
   - commitment carry: 33,405,998;
   - true action charge, including commitment carry: 37,406,126;
   - pass-1 action charge: 4,000,128;
   - explicit fees: 4,000,000.
3. The trader commits the close with two violated bounds:
   - `maxActionChargeAssets` = 37,406,125, which violates ActionCharge (kind 4);
   - `maxExplicitFeesAssets` = 3,999,999, which violates ExplicitFees (kind 5).
4. The public view `assessCommittedOrder` reverts `ConstraintViolation(5 ExplicitFees, 4,000,000, 3,999,999)`.
5. The keeper executes. The order fails terminally with ConstraintViolation, and the authenticated receipt reports `failure.constraint = 5`, actual 4,000,000, limit 3,999,999. The canonical first violation is ActionCharge, 37,406,126 > 37,406,125.

Output of the run:
```
Ran 2 tests for test/PocD_CloseBoundOrder.t.sol:PocD_CloseBoundOrderTest
[PASS] test_D1_CloseBoundFailureReportsNonCanonicalConstraint() (gas: 6257739)
Logs:
  Step 1: open a 300k LONG through the Router and accrue 3 days of carry
  Step 2: dry run of a 10k partial close with unlimited bounds (state is reverted afterwards)
    carry collected at close commitment (atoms): 33405998
    true assessed action charge incl. commitment carry: 37406126
    action charge WITHOUT commitment carry (pass 1): 4000128
    explicit fees: 4000000
  Step 3: commit the same close with two violated bounds:
    maxActionChargeAssets = true charge - 1   -> ActionCharge (kind 4) is violated
    maxExplicitFeesAssets = explicit fees - 1 -> ExplicitFees (kind 5) is violated
  Step 4: the public view CfdOrderPolicyEvaluator.assessCommittedOrder for the pending close
    view reverts ConstraintViolation(kind, actual, limit): 5 4000000 3999999
  Step 5: keeper executes; the order fails terminally and the receipt is emitted
    terminal reason (ConstraintViolation expected): 8
    receipt.failure.constraint (4=ActionCharge, 5=ExplicitFees): 5
    receipt.failure.actual / limit: 4000000 3999999
    canonical first violation would be ActionCharge actual / limit: 37406126 37406125
[PASS] test_D1b_GrossDebitEvidenceOmitsCommitmentCarry() (gas: 5985303)
Logs:
  Step 1: open a 300k LONG through the Router and accrue 6 hours of carry
  Step 2: dry run of a 10k partial close with unlimited bounds (state is reverted afterwards)
      commitment carry: 2783951
      true gross account debit (incl. commitment carry): 6784079
      gross debit seen by pass 1 (without commitment carry): 4000128
  Step 3: commit the close with maxGrossAccountDebitAssets = pass-1 gross - 1, all else unlimited
      (the Book accepts it at commitment: the limit still covers the commitment carry)
      receipt.failure.constraint (3=GrossAccountDebit): 3
      receipt.failure.actual / limit: 4000128 4000127
      real gross account debit of this close (pass-1 actual + commitment carry): 6784079
      understatement of the authenticated evidence (atoms): 2783951
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 13.40ms (13.89ms CPU time)
Ran 1 test suite in 14.51ms (13.40ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

### Full test code (`packages/perps/test/PocD_CloseBoundOrder.t.sol`)
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// D-1: CfdOrderPolicyEvaluator.assessCommittedOrder checks a committed close against its pinned bounds twice:
//   pass 1: _assessSnapshot -> _evaluateClose -> _enforceBounds, WITHOUT the close's commitment carry;
//   pass 2: _includeCommitment adds commitment carry to GrossAccountDebit / ActionCharge, then _enforceBounds again.
// OrderV3Types.ConstraintKind: "Ordering is canonical and is also the required evaluation order."
// When a constraint that comes LATER in that order (ExplicitFees, PostPositionSize, ...) fails in pass 1, the evaluator
// reverts with it, although an EARLIER constraint (GrossAccountDebit / ActionCharge) is violated by the true
// (commitment-inclusive) assessment. The public view and the authenticated terminal receipt both report the wrong
// constraint, actual value and limit.
//   test_D1_CloseBoundFailureReportsNonCanonicalConstraint: ActionCharge (4) violated, receipt says ExplicitFees (5).
//   test_D1b_GrossDebitEvidenceOmitsCommitmentCarry: kind is GrossAccountDebit (3) but the reported actual omits the
//     commitment carry that ACCOUNTING_SPEC says is "included exactly once in gross account debit".
//
// Run (from the repo root, after copying this file to packages/perps/test/):
//   forge test --root packages/perps --match-path test/PocD_CloseBoundOrder.t.sol -vv
// No RPC, env vars or extra dependencies; uses the suite's BasePerpTest fixture and MockPyth.

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdOrderPolicyEvaluator} from "@plether/perps/interfaces/ICfdOrderPolicyEvaluator.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {Vm} from "forge-std/Vm.sol";
import {console} from "forge-std/console.sol";

contract PocD_CloseBoundOrderTest is BasePerpTest {

    address internal constant ACCOUNT = address(0xD1B1);
    address internal constant KEEPER = address(0xD1B2);
    uint256 internal constant MARK = 1e8;
    uint256 internal constant SEED_SIZE = 300_000e18;
    uint256 internal constant SEED_MARGIN = 15_000e6;
    uint256 internal constant CLOSE_SIZE = 10_000e18;
    uint256 internal constant FUNDING = 50_000e6;

    function _request(
        bytes32 clientOrderId,
        bool isClose,
        uint256 maxActionChargeAssets,
        uint256 maxExplicitFeesAssets
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        return _request(clientOrderId, isClose, type(uint256).max, maxActionChargeAssets, maxExplicitFeesAssets);
    }

    function _request(
        bytes32 clientOrderId,
        bool isClose,
        uint256 maxGrossAccountDebitAssets,
        uint256 maxActionChargeAssets,
        uint256 maxExplicitFeesAssets
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientOrderId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = isClose ? CLOSE_SIZE : SEED_SIZE;
        request.marginDelta = isClose ? 0 : SEED_MARGIN;
        request.targetPrice = isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 1 hours),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: maxGrossAccountDebitAssets,
            maxActionChargeAssets: maxActionChargeAssets,
            maxExplicitFeesAssets: maxExplicitFeesAssets,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _receipt(
        uint64 id,
        Vm.Log[] memory logs
    ) internal view returns (OrderV3Types.OrderReceipt memory receipt) {
        IOrderLifecycleBook book = IOrderLifecycleBook(address(router.lifecycleBook()));
        for (uint256 i; i < logs.length; i++) {
            if (
                logs[i].emitter != address(book) || logs[i].topics.length != 4
                    || logs[i].topics[0] != IOrderLifecycleBook.OrderFinalized.selector
            ) {
                continue;
            }
            (,, uint64 terminalTime, OrderV3Types.OrderReceipt memory found) =
                abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
            if (found.orderId != id) {
                continue;
            }
            assertTrue(book.verifyReceipt(found, terminalTime), "receipt must authenticate");
            return found;
        }
        revert("missing receipt");
    }

    function _execute(
        uint64 id
    ) internal returns (OrderV3Types.OrderReceipt memory receipt) {
        bytes[] memory data = _mockPythUpdateData(MARK);
        vm.recordLogs();
        vm.prank(KEEPER);
        router.executeOrder(id, data);
        receipt = _receipt(id, vm.getRecordedLogs());
    }

    function _commit(
        OrderV3Types.OrderRequest memory request
    ) internal returns (uint64 id) {
        vm.prank(ACCOUNT);
        id = router.commitOrder(request);
    }

    function test_D1_CloseBoundFailureReportsNonCanonicalConstraint() public {
        console.log("Step 1: open a 300k LONG through the Router and accrue 3 days of carry");
        _fundTrader(ACCOUNT, FUNDING);
        // Permissionless mark refresh by an untrusted keeper through the public Router entrypoint.
        vm.prank(KEEPER);
        router.updateMarkPrice(_mockPythUpdateData(MARK));
        OrderV3Types.OrderReceipt memory seed =
            _execute(_commit(_request(bytes32(uint256(1)), false, type(uint256).max, type(uint256).max)));
        assertEq(uint8(seed.status), uint8(OrderV3Types.LifecycleStatus.Executed), "seed executed");
        vm.warp(vm.getBlockTimestamp() + 3 days);
        vm.roll(vm.getBlockNumber() + 1);
        // Permissionless mark refresh by an untrusted keeper through the public Router entrypoint.
        vm.prank(KEEPER);
        router.updateMarkPrice(_mockPythUpdateData(MARK));

        console.log("Step 2: dry run of a 10k partial close with unlimited bounds (state is reverted afterwards)");
        uint256 snap = vm.snapshotState();
        OrderV3Types.OrderReceipt memory dry =
            _execute(_commit(_request(bytes32(uint256(2)), true, type(uint256).max, type(uint256).max)));
        assertEq(uint8(dry.status), uint8(OrderV3Types.LifecycleStatus.Executed), "dry close executed");
        uint256 commitCarry = dry.commitment.carryCollectedAssets;
        uint256 actionCharge = dry.economics.actionChargeAssessedAssets; // includes commitment carry
        uint256 explicitFees = dry.economics.executionFeeAssets + dry.economics.frozenSpreadAssets;
        console.log("  carry collected at close commitment (atoms):", commitCarry);
        console.log("  true assessed action charge incl. commitment carry:", actionCharge);
        console.log("  action charge WITHOUT commitment carry (pass 1):", actionCharge - commitCarry);
        console.log("  explicit fees:", explicitFees);
        assertGt(commitCarry, 1, "commitment carry exists");
        assertTrue(vm.revertToState(snap));

        console.log("Step 3: commit the same close with two violated bounds:");
        console.log("  maxActionChargeAssets = true charge - 1   -> ActionCharge (kind 4) is violated");
        console.log("  maxExplicitFeesAssets = explicit fees - 1 -> ExplicitFees (kind 5) is violated");
        uint256 actionLimit = actionCharge - 1;
        uint256 feeLimit = explicitFees - 1;
        // pass-1 action charge must fit the limit so pass 1 reaches ExplicitFees
        assertLe(actionCharge - commitCarry, actionLimit, "pass-1 charge fits the action limit");
        uint64 id = _commit(_request(bytes32(uint256(2)), true, actionLimit, feeLimit));

        console.log("Step 4: the public view CfdOrderPolicyEvaluator.assessCommittedOrder for the pending close");
        vm.warp(vm.getBlockTimestamp() + 1);
        try policyEvaluator.assessCommittedOrder(address(engine), id, KEEPER, MARK) {
            revert("assessment unexpectedly passed");
        } catch (bytes memory reason) {
            assertEq(
                bytes4(reason),
                ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector,
                "constraint violation"
            );
            bytes memory args = new bytes(reason.length - 4);
            for (uint256 i; i < args.length; ++i) {
                args[i] = reason[i + 4];
            }
            (OrderV3Types.ConstraintKind kind, uint256 actual, uint256 limit) =
                abi.decode(args, (OrderV3Types.ConstraintKind, uint256, uint256));
            console.log("  view reverts ConstraintViolation(kind, actual, limit):", uint256(kind), actual, limit);
            assertEq(uint8(kind), uint8(OrderV3Types.ConstraintKind.ExplicitFees), "view reports ExplicitFees");
        }

        console.log("Step 5: keeper executes; the order fails terminally and the receipt is emitted");
        OrderV3Types.OrderReceipt memory receipt = _execute(id);
        console.log("  terminal reason (ConstraintViolation expected):", uint256(receipt.reason));
        console.log("  receipt.failure.constraint (4=ActionCharge, 5=ExplicitFees):", uint256(receipt.failure.constraint));
        console.log("  receipt.failure.actual / limit:", receipt.failure.actual, receipt.failure.limit);
        console.log("  canonical first violation would be ActionCharge actual / limit:", actionCharge, actionLimit);
        assertEq(uint8(receipt.reason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertGt(actionCharge, actionLimit, "ActionCharge (kind 4) really is violated");
        assertEq(
            uint8(receipt.failure.constraint),
            uint8(OrderV3Types.ConstraintKind.ExplicitFees),
            "receipt reports the LATER constraint although ActionCharge is violated first in canonical order"
        );
        assertEq(receipt.failure.actual, explicitFees, "receipt actual is the fee, not the action charge");
        assertEq(receipt.failure.limit, feeLimit, "receipt limit is the fee limit");
    }


    function _openAndAccrue(
        uint256 accrual
    ) internal {
        _fundTrader(ACCOUNT, FUNDING);
        // Permissionless mark refresh by an untrusted keeper through the public Router entrypoint.
        vm.prank(KEEPER);
        router.updateMarkPrice(_mockPythUpdateData(MARK));
        OrderV3Types.OrderReceipt memory seed =
            _execute(_commit(_request(bytes32(uint256(1)), false, type(uint256).max, type(uint256).max)));
        assertEq(uint8(seed.status), uint8(OrderV3Types.LifecycleStatus.Executed), "seed executed");
        vm.warp(vm.getBlockTimestamp() + accrual);
        vm.roll(vm.getBlockNumber() + 1);
        // Permissionless mark refresh by an untrusted keeper through the public Router entrypoint.
        vm.prank(KEEPER);
        router.updateMarkPrice(_mockPythUpdateData(MARK));
    }

    /// Same root cause, second facet: even when the reported KIND is right (GrossAccountDebit), the reported ACTUAL
    /// value excludes the commitment carry that ACCOUNTING_SPEC says is "included exactly once in gross account debit".
    function test_D1b_GrossDebitEvidenceOmitsCommitmentCarry() public {
        console.log("Step 1: open a 300k LONG through the Router and accrue 6 hours of carry");
        _openAndAccrue(6 hours);

        console.log("Step 2: dry run of a 10k partial close with unlimited bounds (state is reverted afterwards)");
        uint256 snap = vm.snapshotState();
        OrderV3Types.OrderReceipt memory dry =
            _execute(_commit(_request(bytes32(uint256(2)), true, type(uint256).max, type(uint256).max)));
        uint256 commitCarry = dry.commitment.carryCollectedAssets;
        uint256 grossTrue = dry.economics.grossAccountDebitAssets; // includes commitment carry
        console.log("    commitment carry:", commitCarry);
        console.log("    true gross account debit (incl. commitment carry):", grossTrue);
        console.log("    gross debit seen by pass 1 (without commitment carry):", grossTrue - commitCarry);
        assertTrue(vm.revertToState(snap));

        console.log("Step 3: commit the close with maxGrossAccountDebitAssets = pass-1 gross - 1, all else unlimited");
        console.log("    (the Book accepts it at commitment: the limit still covers the commitment carry)");
        uint256 grossLimit = grossTrue - commitCarry - 1;
        assertGe(grossLimit, commitCarry, "limit covers commitment carry, so commitment is accepted");
        uint64 id = _commit(_request(bytes32(uint256(2)), true, grossLimit, type(uint256).max, type(uint256).max));
        OrderV3Types.OrderReceipt memory receipt = _execute(id);
        console.log("    receipt.failure.constraint (3=GrossAccountDebit):", uint256(receipt.failure.constraint));
        console.log("    receipt.failure.actual / limit:", receipt.failure.actual, receipt.failure.limit);
        console.log("    real gross account debit of this close (pass-1 actual + commitment carry):",
            receipt.failure.actual + commitCarry);
        console.log("    understatement of the authenticated evidence (atoms):", commitCarry);
        assertEq(uint8(receipt.reason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(uint8(receipt.failure.constraint), uint8(OrderV3Types.ConstraintKind.GrossAccountDebit));
        assertEq(receipt.failure.limit, grossLimit, "limit");
        // reported actual is the pass-1 value: far below the true commitment-inclusive gross debit
        assertLe(receipt.failure.actual, grossTrue - commitCarry + 10_000, "reported actual is the pass-1 value");
        assertLt(receipt.failure.actual, grossTrue - commitCarry / 2, "reported actual omits the commitment carry");
    }
}
```
