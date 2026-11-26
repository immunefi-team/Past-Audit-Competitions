# \#98198 [SC-Low] Carry collected when an increase order is committed bypasses the trader's pinned maxActionChargeAssets / maxGrossAccountDebitAssets and is missing from the commitment receipt

**Submitted on Nov 5th 2026 at 22:19:06 UTC by @Lester for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98198
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
Every fresh order pins inclusive financial limits, `maxActionChargeAssets` and `maxGrossAccountDebitAssets`. The spec says "Receipts and financial bounds include the direct collection [of carry] once". For a close this holds: carry collected when the order is committed is recorded in the lifecycle commitment and checked against the bounds.

For an **increase** it does not hold. `commitOrder` collects all of the position's pending carry while it reserves the execution bounty. That carry is recorded nowhere (the commitment shows 0 / 0 / 0) and checked against nothing. So a trader's pinned action-charge and gross-debit limits do not bound what the order actually charges.

In the PoC the trader pins a 4.0 USDG action-charge limit. The order is charged 37.4 USDG, 9.35x the bound. The protocol's own public evaluator rejects the same order before commitment, yet the Router executes it.

## Vulnerability Details
**Root cause**

1. `router/OrderReservationAccounting.sol:108-128`, `_reserveExecutionBounty` (open branch):
   - It returns `effects` with only `bountyFromFreeAssets` set.
   - It then calls `clearinghouse.lockReservedSettlement(account, bounty)`.
2. `MarginClearinghouse.sol:818-824`, `lockReservedSettlement`:
   - It runs `_checkpointCarryBeforeMarginChange(account)`, which goes through `CfdEngine.realizeCarryBeforeMarginChange` to `_realizeCarryFromSettlement`.
   - That debits ALL pending carry of the existing position (pledge first, then free settlement) and pays it to the pool.
   - `reserveCommittedOrderMargin` (`MarginClearinghouse.sol:620-633`) checkpoints again.
3. `OrderLifecycleBook.sol:327-361`, `recordCommitment`:
   - For an increase it stores `carryCollectedAssets = 0`, `settlementBeforeAssets = 0` and `settlementAfterAssets = 0`.
   - It checks nothing against `maxActionChargeAssets` / `maxGrossAccountDebitAssets` beyond the bounty.
4. `CfdOrderPolicyEvaluator.sol:63-65`:
   - At execution, commitment carry is added to the assessment (`_includeCommitment`) only `if (order.isClose)`.
   - The increase assessment uses only the carry accrued after the commit-time checkpoint.

**Spec vs code**

- ACCOUNTING_SPEC.md:723 (LP-capital carry): "realized carry is booked as LP trading revenue once. Receipts and financial bounds include the direct collection once, in addition to the residual action settlement".
- `OrderV3Types.sol:133` (ExecutionBounds): "`maxGrossAccountDebitAssets` covers settlement debit, trader-claim consumption, and the reserved bounty".
- SECURITY.md (Bounded authority): "Every fresh order pins ... inclusive financial limits".
- `OrderLifecycleBook.sol:98` (StoredCommitment): "Pre-commitment custody is exactly post-commitment custody plus collected carry". For an increase this is false: both custody values are recorded as 0 while carry was debited.
- The close path does exactly what the spec says. `recordCommitment` bounds close-commitment carry, and `_includeCommitment` re-adds it at execution; the control test shows a close with the same carry reverting `OrderLifecycleBook__CommitmentBoundExceeded`.
- The evaluator is designed to include an increase's pre-existing carry. The repo tests `CfdOrderPolicyEvaluator.t.sol::test_IncreaseReportsCarryActuallyRealizedBeforeOpenCost` and `CfdOrderPolicyEvaluatorParity.t.sol::test_AssessIncreaseAfterCarryMatchesDirectSidecarSnapshotPlanAndEvaluation` expect `carryAssets`, `actionChargeAssessedAssets` and `grossAccountDebitAssets` of an increase to include accrued carry. Through the real Router that carry never reaches the assessment, because `commitOrder` has already collected it.

**Not a duplicate of known issues**

- Cecuro LOW-34 is the opposite case on the close path: assessments over-reporting pending carry. Different root cause.
- MED-30's note concerns close limits on cancel-and-close.
- The AI-scan carry items concern rounding and depth.
- No public plether-core commit touches open/increase commitment carry.

## Impact Details
**Impact (Low):** "Contract fails to deliver promised returns, but doesn't lose value". The code does not work as intended:
- the order-level financial limits a trader signs do not bound the carry that the order itself collects;
- the authenticated lifecycle receipt under-reports the order's commitment debit (0 instead of 33.4 USDG in the PoC).

Integrators and traders that rely on `maxActionChargeAssets` / `maxGrossAccountDebitAssets`, or on the receipt, to cap or audit what an order costs get a wrong guarantee. The protocol's own `CfdOrderPolicyEvaluator.assessOrder` refuses an order that the Router then executes.

**Concrete scenario:** consider an integrator contract that submits orders on behalf of its users. It reserves each user's funds per order according to `maxGrossAccountDebitAssets` and reconciles its books from the authenticated receipt. Its books end up off by the full pending carry of the position. In the PoC that is 33.4 USDG on an order whose pinned gross-debit limit is 4.2 USDG, and the receipt reports 0 carry collected.

No funds are stolen, because the carry is owed. That is exactly why this is filed as Low. It is not an Insight, for three reasons:
- the signed limit is actually exceeded in execution (by 9.35x), not just theoretically weak;
- the protocol's own evaluator rejects the order that the Router then executes;
- the behaviour is an asymmetry with the close path, which records and bounds the same carry. It is not a missing feature.

Preconditions: any account with an open position and accrued carry (any nonzero `baseCarryBps`; release value 500 bps) that commits an increase.

**Suggested fix:** do either of the following:
- record open/increase commitment carry in the commitment (before/after custody plus `carryCollected`), check it in `recordCommitment`, and apply `_includeCommitment` to opens as well, as on the close path;
- or do not checkpoint carry at increase commitment, so the evaluator's execution-time assessment includes it.

## References
- Code (frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`):
  - `packages/perps/src/router/OrderReservationAccounting.sol#L108-L128`
  - `packages/perps/src/MarginClearinghouse.sol#L818-L824`, `#L620-L633`
  - `packages/perps/src/OrderLifecycleBook.sol#L98`, `#L327-L361`
  - `packages/perps/src/CfdOrderPolicyEvaluator.sol#L63-L65`
  - `packages/perps/src/OrderV3Types.sol#L133`
- Spec: `packages/perps/ACCOUNTING_SPEC.md#L723`; `packages/perps/SECURITY.md` (Bounded authority).

## Link to Proof of Concept

https://gist.github.com/Adam-Pejski-work/b24c310fd8efa4f8db3e9b4e995cbaa4

## Proof of Concept

## Proof of Concept
A runnable Foundry test in the repository's own perps test suite, at the frozen commit, with no change to any in-scope contract. It uses the repo fixture `BasePerpTest`, whose MockPyth and mock USDG are the allowed out-of-scope mocks.

Every action goes through public entrypoints:
- the trader calls `router.commitOrder`;
- an untrusted keeper calls `router.executeOrder` and the permissionless `router.updateMarkPrice`.

The full test code is included below; it is also in a secret gist: https://gist.github.com/Adam-Pejski-work/b24c310fd8efa4f8db3e9b4e995cbaa4.

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the test code below as packages/perps/test/PocC1_IncreaseCommitCarry.t.sol
forge test --root packages/perps --match-path test/PocC1_IncreaseCommitCarry.t.sol -vv   # Foundry 1.5.1
```

No env or RPC is needed. Expected: `2 passed`.

**`test_IncreaseCommitmentCarryEscapesBoundsAndReceipt`** (amounts in USDG atoms, 6 decimals):
1. The trader opens a 300k LONG and 3 days of carry accrue: 33,405,998 of pending carry.
2. A dry run (state snapshot) measures the execution-time charge of a 10k increase: action charge 4,000,128 and gross debit 4,200,128. The trader pins exactly these as `maxActionChargeAssets` / `maxGrossAccountDebitAssets`.
3. The protocol's public `CfdOrderPolicyEvaluator.assessOrder` rejects this order and bounds: `ConstraintViolation(GrossAccountDebit, actual 37,605,998, limit 4,200,128)`.
4. `commitOrder` with these bounds succeeds:
   - it debits 33,405,998 of carry outside every bound;
   - the receipt commitment shows `carryCollectedAssets = 0`, `settlementBeforeAssets = 0`, `settlementAfterAssets = 0`.
5. The order **executes**:
   - action charge actually paid for this order: 37,406,126, against the pinned 4,000,128 (9.35x);
   - total account debit: 37,606,126, against the pinned 4,200,128;
   - gas for commit + execute: 2,607,166.

**`test_Control_CloseCommitmentCarryIsBounded`:** the same carry collected by a partial-close commitment is recorded in the Book. When the bound is one atom lower, the commit reverts with `OrderLifecycleBook__CommitmentBoundExceeded`. This isolates the increase path as the defect.

Output of the run (abridged):
```
[PASS] test_Control_CloseCommitmentCarryIsBounded()
  Control: close commit reverted with OrderLifecycleBook__CommitmentBoundExceeded
[PASS] test_IncreaseCommitmentCarryEscapesBoundsAndReceipt()
  Step 3: CfdOrderPolicyEvaluator.assessOrder rejects it: constraint 3 (3 = GrossAccountDebit) actual / limit: 37605998 4200128
  Step 4: ... carry debited by commitOrder (outside every bound), atoms: 33405998
    receipt.commitment.carryCollectedAssets              : 0
    total action charge paid for this order (commit+exec): 37406126
    maxActionChargeAssets pinned by trader               : 4000128
    total account debit for this order                   : 37606126
    maxGrossAccountDebitAssets pinned by trader          : 4200128
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

### Full test code (`packages/perps/test/PocC1_IncreaseCommitCarry.t.sol`)
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// C1-L1 PoC: carry collected when an INCREASE order is committed escapes the order's pinned financial bounds
// (maxActionChargeAssets / maxGrossAccountDebitAssets) and is missing from the lifecycle commitment and receipt.
// The same carry collected by a CLOSE commitment is recorded and bounded (control case at the end).
//
// Run (from packages/perps, after copying this file to packages/perps/test/):
//   forge test --match-path test/PocC1_IncreaseCommitCarry.t.sol -vv
// No env or RPC needed. Uses the repo's BasePerpTest fixture (MockPyth, MockSettlementToken) without changes to src.

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdOrderPolicyEvaluator} from "@plether/perps/interfaces/ICfdOrderPolicyEvaluator.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {Vm} from "forge-std/Vm.sol";
import {console} from "forge-std/console.sol";

contract PocC1_IncreaseCommitCarryTest is BasePerpTest {

    address internal constant ACCOUNT = address(0xC1A1);
    address internal constant KEEPER = address(0xC1A2);
    uint256 internal constant MARK = 1e8;
    uint256 internal constant SEED_SIZE = 300_000e18; // 3,000 lots, ~$300k notional
    uint256 internal constant SEED_MARGIN = 15_000e6;
    uint256 internal constant STEP_SIZE = 10_000e18; // 100 lots, ~$10k notional
    uint256 internal constant STEP_MARGIN = 1000e6;
    uint256 internal constant FUNDING = 50_000e6;
    uint256 internal constant CARRY_ELAPSED = 3 days;

    function _request(
        bytes32 clientOrderId,
        bool isClose,
        uint256 maxGrossAccountDebitAssets,
        uint256 maxActionChargeAssets
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientOrderId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = isClose ? STEP_SIZE : (clientOrderId == bytes32(uint256(1)) ? SEED_SIZE : STEP_SIZE);
        request.marginDelta = isClose ? 0 : (clientOrderId == bytes32(uint256(1)) ? SEED_MARGIN : STEP_MARGIN);
        // Unbounded slippage, same convention as test/utils/LegacyOrderRouterHarness.sol for a LONG side.
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
            maxExplicitFeesAssets: type(uint256).max,
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

    function _tail(
        bytes memory err
    ) internal pure returns (bytes memory args) {
        args = new bytes(err.length - 4);
        for (uint256 i; i < args.length; i++) {
            args[i] = err[i + 4];
        }
    }

    function _commitAndExecute(
        OrderV3Types.OrderRequest memory request
    ) internal returns (uint64 id, uint256 commitDebit, OrderV3Types.OrderReceipt memory receipt) {
        uint256 balanceBeforeCommit = clearinghouse.balanceAssets(ACCOUNT);
        vm.prank(ACCOUNT);
        id = router.commitOrder(request);
        commitDebit = balanceBeforeCommit - clearinghouse.balanceAssets(ACCOUNT);
        bytes[] memory data = _mockPythUpdateData(MARK);
        vm.recordLogs();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        receipt = _receipt(id, vm.getRecordedLogs());
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    /// @dev Seeds a LONG position through the public V3 router, then lets carry accrue for three days.
    function _seedPositionAndAccrueCarry() internal {
        _fundTrader(ACCOUNT, FUNDING);
        // Permissionless mark refresh by an untrusted keeper through the public Router entrypoint.
        vm.prank(KEEPER);
        router.updateMarkPrice(_mockPythUpdateData(MARK));
        _commitAndExecute(_request(bytes32(uint256(1)), false, type(uint256).max, type(uint256).max));
        (uint256 size,,,,,,) = engine.positions(ACCOUNT);
        assertEq(size, SEED_SIZE, "seed position open");

        vm.warp(vm.getBlockTimestamp() + CARRY_ELAPSED);
        vm.roll(vm.getBlockNumber() + 1);
        // Permissionless mark refresh by an untrusted keeper through the public Router entrypoint.
        vm.prank(KEEPER);
        router.updateMarkPrice(_mockPythUpdateData(MARK));
    }

    function test_IncreaseCommitmentCarryEscapesBoundsAndReceipt() public {
        console.log("Step 1: trader opens a 300k LONG via the router, then 3 days of LP carry accrue");
        _seedPositionAndAccrueCarry();
        uint256 pendingCarry = _expectedIndexedCarryAssets(ACCOUNT);
        console.log("  pending (uncollected) carry on the position, USDG atoms:", pendingCarry);
        assertGt(pendingCarry, 0, "carry accrued");

        // Step 2: dry run (snapshot) to learn the exact execution-time charge of a 10k increase.
        uint256 snap = vm.snapshotState();
        (, uint256 dryCommitDebit, OrderV3Types.OrderReceipt memory dry) =
            _commitAndExecute(_request(bytes32(uint256(2)), false, type(uint256).max, type(uint256).max));
        uint256 execActionCharge = dry.economics.actionChargeAssessedAssets;
        uint256 execGrossDebit = dry.economics.grossAccountDebitAssets;
        console.log("Step 2 (dry run): execution-time assessed action charge:", execActionCharge);
        console.log("                  execution-time assessed gross debit :", execGrossDebit);
        console.log("                  carry silently debited at commitment:", dryCommitDebit);
        assertTrue(vm.revertToState(snap));

        // Step 3: the protocol's own public evaluator, asked BEFORE commitment, says these bounds reject this
        // increase, because it counts the position's pending carry as part of the order's gross debit / charge.
        OrderV3Types.OrderRequest memory tight = _request(bytes32(uint256(2)), false, execGrossDebit, execActionCharge);
        CfdTypes.Order memory prospective = CfdTypes.Order({
            account: ACCOUNT,
            sizeDelta: STEP_SIZE,
            marginDelta: STEP_MARGIN,
            targetPrice: tight.targetPrice,
            commitTime: uint64(vm.getBlockTimestamp()),
            commitBlock: uint64(vm.getBlockNumber()),
            orderId: router.nextCommitId(),
            side: CfdTypes.Side.LONG,
            isClose: false
        });
        try policyEvaluator.assessOrder(
            address(engine), prospective, KEEPER, MARK, pool.totalAssets(), tight.bounds, dry.bountyAssets
        ) returns (
            OrderV3Types.ExecutionAssessment memory
        ) {
            revert("evaluator should reject the pinned bounds");
        } catch (bytes memory err) {
            assertEq(
                bytes4(err),
                ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector,
                "public evaluator rejects the increase under the pinned bounds"
            );
            (uint8 kind, uint256 actual, uint256 limit) = abi.decode(_tail(err), (uint8, uint256, uint256));
            console.log("Step 3: CfdOrderPolicyEvaluator.assessOrder rejects it: constraint", uint256(kind));
            console.log("        (3 = GrossAccountDebit) actual / limit:", actual, limit);
            assertEq(kind, uint8(OrderV3Types.ConstraintKind.GrossAccountDebit), "gross debit including carry");
        }

        // Step 4: the trader commits exactly those bounds through the Router (equality passes at execution).
        console.log("Step 4: commit the same increase with maxActionCharge/maxGrossDebit pinned at those values");
        uint256 balanceBefore = clearinghouse.balanceAssets(ACCOUNT);
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        uint256 gasBefore = gasleft();
        (uint64 id, uint256 commitDebit, OrderV3Types.OrderReceipt memory receipt) = _commitAndExecute(tight);
        uint256 gasUsed = gasBefore - gasleft();
        uint256 balanceAfter = clearinghouse.balanceAssets(ACCOUNT);
        uint256 keeperCredit = clearinghouse.balanceAssets(KEEPER) - keeperBefore;

        console.log("  carry debited by commitOrder (outside every bound), atoms:", commitDebit);
        console.log("  receipt.commitment.carryCollectedAssets              :", receipt.commitment.carryCollectedAssets);
        console.log("  receipt.commitment.settlementBeforeAssets            :", receipt.commitment.settlementBeforeAssets);
        console.log("  receipt.commitment.settlementAfterAssets             :", receipt.commitment.settlementAfterAssets);
        console.log("  receipt.economics.actionChargeAssessedAssets         :", receipt.economics.actionChargeAssessedAssets);
        console.log("  maxActionChargeAssets pinned by trader               :", execActionCharge);
        console.log("  total action charge paid for this order (commit+exec):", commitDebit + receipt.economics.actionChargeCollectedAssets);
        console.log("  settlement balance before commit / after execution   :", balanceBefore, balanceAfter);
        console.log("  total account debit for this order                   :", balanceBefore - balanceAfter);
        console.log("  maxGrossAccountDebitAssets pinned by trader          :", execGrossDebit);
        console.log("  keeper credit (bounty)                               :", keeperCredit);
        console.log("  gas used (commit + execute)                          :", gasUsed);

        assertGt(commitDebit, 0, "increase commitment collected carry");
        assertEq(receipt.commitment.carryCollectedAssets, 0, "Book records no commitment carry for the increase");
        assertEq(receipt.commitment.settlementBeforeAssets, 0, "Book records no pre-commitment custody");
        assertEq(receipt.commitment.settlementAfterAssets, 0, "Book records no post-commitment custody");
        assertEq(uint8(receipt.reason), uint8(OrderV3Types.TerminalReason.Executed), "executed");
        assertGt(
            commitDebit + receipt.economics.actionChargeCollectedAssets,
            execActionCharge,
            "trader paid more action charge for this order than maxActionChargeAssets"
        );
        assertGt(balanceBefore - balanceAfter, execGrossDebit, "account debit exceeds maxGrossAccountDebitAssets");
        id;
    }

    /// @dev Control: a close commitment collects the same carry, records it in the Book and enforces the bound.
    function test_Control_CloseCommitmentCarryIsBounded() public {
        _seedPositionAndAccrueCarry();
        uint256 pendingCarry = _expectedIndexedCarryAssets(ACCOUNT);
        console.log("Control: pending carry before a partial close commit:", pendingCarry);

        // Limit one atom below the commitment carry: the close commit must revert with CommitmentBoundExceeded.
        OrderV3Types.OrderRequest memory request =
            _request(bytes32(uint256(3)), true, type(uint256).max, pendingCarry - 1);
        vm.prank(ACCOUNT);
        try router.commitOrder(request) returns (uint64) {
            revert("close commit should be bounded");
        } catch (bytes memory err) {
            bytes4 selector = bytes4(err);
            assertEq(
                selector,
                IOrderLifecycleBook.OrderLifecycleBook__CommitmentBoundExceeded.selector,
                "close commitment carry is checked against maxActionChargeAssets"
            );
            console.log("Control: close commit reverted with OrderLifecycleBook__CommitmentBoundExceeded");
        }

        // With an adequate bound, the Book records the close commitment carry.
        request = _request(bytes32(uint256(4)), true, type(uint256).max, type(uint256).max);
        vm.prank(ACCOUNT);
        uint64 id = router.commitOrder(request);
        OrderV3Types.PendingIntent memory pending = router.lifecycleBook().pendingIntent(id);
        console.log("Control: Book commitment carry for the close:", pending.commitment.carryCollectedAssets);
        assertEq(pending.commitment.carryCollectedAssets, pendingCarry, "close commitment carry recorded");
    }

}
```
