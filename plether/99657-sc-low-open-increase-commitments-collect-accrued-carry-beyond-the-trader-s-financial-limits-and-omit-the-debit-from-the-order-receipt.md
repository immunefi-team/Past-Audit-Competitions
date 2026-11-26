# \#99657 [SC-Low] Open/increase commitments collect accrued carry beyond the trader's financial limits and omit the debit from the order receipt

**Submitted on Nov 14th 2026 at 17:24:10 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99657
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief / Introduction

I found that committing an open or position-increase order can collect the carry owed on an existing position without including that collection in the order's financial limits or execution receipt.

Every V3 order has a set of mandatory financial limits supplied by the trader. Those limits are meant to constrain the settlement debit and action charges attributable to the order. For a close, carry collected at commitment is recorded, checked against the limits and included in the later assessment. For an open or increase, the same account can be charged carry while the Router locks the execution bounty or committed margin, but the resulting commitment record still reports zero carry. The execution assessment then leaves that earlier debit out as well.

I reproduced the difference against the frozen competition commit with the production Router, Engine, Clearinghouse, lifecycle book and policy evaluator. An increase with `maxGrossAccountDebitAssets = 365.005048 USDG` ended `Executed` after the account paid **1,705.008966 USDG** during that order's commitment and execution. Of that amount, **1,522.555596 USDG** was collected as carry at commitment, while the final receipt reported only **182.453370 USDG** of gross debit and zero commitment carry.

The discrepancy is not a rounding error or an incorrect carry calculation. The carry was owed and the pool received it. The defect is that the trader's signed debit limits did not constrain the collection associated with the order, and the order's receipt does not account for that collection. The same limits correctly reject a comparable partial-close commitment on an account with accrued carry.

## Vulnerability Details

### 1. The trader's debit limit is mandatory financial authority

`OrderV3Types.ExecutionBounds` is described as mandatory financial authority for a V3 order. In particular, the documented gross-debit ceiling covers settlement debits, consumed trader claims and the reserved execution bounty. `ACCOUNTING_SPEC.md` also states that realized carry is included once in the financial bounds and receipts, in addition to any remaining settlement required by the action.

An increase is an ordinary V3 order. It is not exempted from these bounds merely because some of its settlement effects occur during commitment instead of execution.

### 2. The open commitment collects carry before the order is executed

`OrderCommitHandler._createPendingOrder()` performs the following operations:

```solidity
CfdEnginePlanTypes.CloseCommitment memory effects =
    _reserveExecutionBounty(account, order.sizeDelta, executionBountyAssets, order.isClose);
clearinghouse.recordBountyReservation(
    account, IMarginClearinghouse.BountyKind.Order, orderId, executionBountyAssets
);
_onOrderCommitted(order, effects);
_reserveCommittedMargin(account, orderId, order.isClose, order.marginDelta);
```

The open/increase branch of `_reserveExecutionBounty()` returns commitment effects whose `carryCollectedAssets` and settlement-before/after fields are still zero. When a nonzero bounty is reserved, the branch calls `clearinghouse.lockReservedSettlement(account, bounty)`.

`MarginClearinghouse.lockReservedSettlement()` first calls `_checkpointCarryBeforeMarginChange(account)`. That checkpoint invokes `CfdEngine.realizeCarryBeforeMarginChange()` and collects outstanding carry on the existing position. The collection uses position pledge and eligible free settlement, and the collected amount is transferred to the LP pool.

A second checkpoint is reachable through `MarginClearinghouse.reserveCommittedOrderMargin()`. In the open commitment flow, the Router calls this margin-reservation function **after** `_onOrderCommitted()` has already recorded the commitment effects. Neither checkpointed amount is captured in the open-order commitment record.

### 3. Commitment validation sees zero carry instead of the amount paid

`OrderLifecycleBook.recordCommitment()` checks commitment effects against the gross-debit and action-charge limits:

```solidity
uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;
if (gross > pending.bounds.maxGrossAccountDebitAssets) {
    revert OrderLifecycleBook__CommitmentBoundExceeded(
        OrderV3Types.ConstraintKind.GrossAccountDebit,
        gross,
        pending.bounds.maxGrossAccountDebitAssets
    );
}
if (effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets) {
    revert OrderLifecycleBook__CommitmentBoundExceeded(
        OrderV3Types.ConstraintKind.ActionCharge,
        effects.carryCollectedAssets,
        pending.bounds.maxActionChargeAssets
    );
}
```

Those checks work for closes because the close reservation returns the amount collected. For opens, the commitment effects still contain `carryCollectedAssets = 0`, so even a material carry debit is invisible to these checks. The settlement consistency check does not detect it either because both settlement fields in the open commitment effects are zero.

### 4. The committed-order evaluator and receipt do not recover the omission

`CfdOrderPolicyEvaluator.assessCommittedOrder()` includes previously collected commitment carry only for close orders:

```solidity
assessment = _assessSnapshot(context, order, pending.bounds, planner, snapshot);
if (order.isClose) {
    _includeCommitment(
        assessment,
        pending.commitment,
        pending.bounds,
        pending.executionBountyAssets
    );
}
```

For an increase, the evaluator sees that most carry has already been collected from the account. It therefore assesses only the small remaining carry and the current execution costs. Because it does not include the earlier collection, the final order economics also omit it.

The opposite happens when I call the evaluator before commitment: while the carry is still pending, the open assessment counts that pending carry in its gross debit and action charge. The very same increase can therefore fail the hypothetical pre-commit assessment, commit successfully, and finish `Executed` with an apparently compliant receipt.

### 5. The close path provides the missing control

For closes, `_reserveCloseExecutionBounty()` records the commitment-time carry collection and the lifecycle book checks it against the same bounds. The close evaluator also includes that recorded collection in its final economics.

In my test, Bob has the same old position and accrued carry as Alice. I give his partial close the same `365.005048 USDG` gross limit. Bob's commitment reverts with `CommitmentBoundExceeded(GrossAccountDebit)` because the carry component is `1,523.802157 USDG`. His account is not debited.

Alice's increase, by contrast, pays `1,522.555596 USDG` at commitment and still executes. This isolates the missing commitment accounting to the open/increase path rather than the underlying carry mechanism.

## Reproduction and Measurements

I used the release risk parameters, a 5,000,000 USDG junior pool and a constant basket price of `1.0000`. Alice, Bob, Carol and Dave each open a 400,000-unit LONG with 10,000 USDG of pledged margin, then accrue carry for 91 days. The test proceeds through real V3 order commitment and execution entrypoints; only Pyth and USDG are supplied by the repository's existing test fixture.

The gross-debit and action-charge ceilings are twice the evaluator's quote for a 50,000-unit increase on a carry-free comparison account:

| Value | Amount (USDG) |
|---|---:|
| Gross-debit limit | 365.005048 |
| Action-charge limit | 364.605048 |
| Alice's commitment-time carry debit | 1,522.555596 |
| Alice's execution debit | 182.453370 |
| **Alice's combined debit** | **1,705.008966** |
| **Amount above the gross-debit limit** | **1,340.003918** |
| Gross debit in Alice's execution receipt | 182.453370 |
| `commitment.carryCollectedAssets` in receipt | 0 |

Alice's combined debit is approximately **4.67 times** her gross limit. The test also confirms that the entire commitment-time debit comes out of the existing position pledge and arrives as cash in the LP pool.

I used three comparison accounts to check the source of the discrepancy:

| Account | Control | Result |
|---|---|---|
| Alice | Increase with pending carry and tight limits | Executes after collecting 1,705.008966 USDG in total; receipt shows 182.453370 USDG |
| Bob | Partial close with the same limits and accrued carry | Commitment reverts; no debit |
| Carol | Pays carry through an ordinary owner deposit before the same increase | Increase executes with 187.440557 USDG of order-associated debit, inside the limit |
| Dave | Hypothetical increase assessed before commitment | Assessment rejects 1,708.746532 USDG gross debit; the subsequent commitment and execution succeed |

I also tested how this affects limits derived directly from an evaluator quote. A carry-free account received a quoted action charge of `102.444026 USDG`, with a 25% tolerance ceiling of `128.055032 USDG`. A larger order ahead in the queue increased the eventual trading charge, and this carry-free account's increase failed on its limit.

The otherwise comparable account with pending carry received a quote that included `863.844979 USDG` of action charges, including the old carry. The quote-derived ceiling was `1,079.806223 USDG`. Commitment collected the old carry without recording it against that ceiling, leaving sufficient room for the increase to execute at a trading charge of `162.344064 USDG`, or about **158% of the carry-free trade quote**. That control demonstrates how the inconsistent quote and commitment treatment changes a trader's effective tolerance for the execution itself.

## Impact Details

The V3 bounds are an authorization mechanism: they express how much account settlement the trader is prepared to spend on an order. An increase that can collect more than this authorization allows, then produce an execution receipt that reports only the smaller execution-stage debit, does not honor that financial interface.

This matters when an account owner or a trading system uses the V3 bounds as a per-order budget. The order may finish successfully even though the combined debit caused by its commitment and execution is beyond the submitted limit. The final lifecycle assessment and receipt do not provide the full amount needed to reconcile that authorization against the account's actual settlement movement.

The funds remain accounted for within the protocol: carry due on the existing position is paid to the LP pool. The impact is the failure to enforce the promised financial limits and report the complete order-associated settlement effects, not an assertion that the carry debt is fabricated or that funds have been stolen.

## Suggested Remediation

Collect or measure carry before finalizing the open/increase commitment effects, then record the amount through the same bound-checking path used for close commitments. The implementation needs to cover both the opening bounty lock and the committed-margin reservation, including the case in which the bounty is zero but the margin reservation still checkpoints carry.

`recordCommitment()` must receive the actual commitment-time collection and enforce the user's limits atomically, reverting the entire commitment when those limits are exceeded. The committed-order evaluator should then incorporate recorded commitment carry for increases as well as closes, exactly once, so the execution assessment, action-charge bounds and final receipt remain consistent with the account's settlement movement.

## References

- [OrderV3Types.sol - mandatory financial authority and debit limit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderV3Types.sol#L131-L149)
- [OrderCommitHandler.sol - commitment order of operations](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L21-L35)
- [OrderReservationAccounting.sol - open bounty reservation and missing effects](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L108-L144)
- [MarginClearinghouse.sol - committed margin reservation checkpoints carry](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L615-L638)
- [MarginClearinghouse.sol - reserved settlement lock checkpoints carry](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L812-L825)
- [CfdEngine.sol - carry realization before margin changes](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L535-L550)
- [OrderLifecycleBook.sol - commitment debit checks](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L327-L363)
- [CfdOrderPolicyEvaluator.sol - commitment carry included only for closes](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluator.sol#L46-L69)
- [CfdOrderPolicyEvaluatorBase.sol - pending carry enters open-order assessment](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L330-L351)
- [ACCOUNTING_SPEC.md - realized carry and financial-bound treatment](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L711-L726)



## Proof of Concept

## Proof of Concept


## Reproduction

I ran this test against commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` using Foundry 1.5.1 and Solidity 0.8.35. The only mocks are those already supplied by the repository's `BasePerpTest` fixture. I used ordinary `commitOrder`, `executeOrder`, `updateMarkPrice` and Clearinghouse deposit calls. I did not change the source contracts or patch protocol storage.

Save the source below as:

```text
packages/perps/test/perps/insight/OpenCommitCarryOutsideBounds.t.sol
```

From the repository root, run:

```bash
cd /workspaces/audit-comp-plether
export PATH="$HOME/.foundry/bin:$PATH"

git rev-parse HEAD

mkdir -p packages/perps/test/perps/bounds_only
mkdir -p carry_bounds_evidence

cp   packages/perps/test/perps/insight/OpenCommitCarryOutsideBounds.t.sol   packages/perps/test/perps/bounds_only/OpenCommitCarryOutsideBounds.t.sol

sha256sum packages/perps/test/perps/bounds_only/OpenCommitCarryOutsideBounds.t.sol

FOUNDRY_TEST="test/perps/bounds_only" forge test   --root packages/perps   --offline   -j 1   --skip script   --match-path 'test/perps/bounds_only/OpenCommitCarryOutsideBounds.t.sol'   -vv   2>&1 | tee carry_bounds_evidence/FINAL_ALL_TESTS.log

RESULT=${PIPESTATUS[0]}
echo "TEST EXIT STATUS: $RESULT"

grep -E '\[PASS\]|\[FAIL\]|Suite result|ALICE|BOB|CAROL|DAVE|quoted trade charge'   carry_bounds_evidence/FINAL_ALL_TESTS.log || true

git diff HEAD -- packages/perps/src
```

Both tests ran successfully using this isolated test-directory configuration. This keeps the test's normal imports and uses the original production sources. The earlier full-directory compiler invocation was terminated before test execution; the isolated invocation compiled and completed successfully.

## Complete Solidity Test Source

SHA-256: `9d060a1b1dfc54c4a027a2d2288314a766058e947bd3aac61566991a298123e7`  
Source lines: 565

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdOrderPolicyEvaluator} from "@plether/perps/interfaces/ICfdOrderPolicyEvaluator.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Carry collected by the commit of an open / increase order is outside the order's
///         `maxGrossAccountDebitAssets` and `maxActionChargeAssets`, and is missing from its receipt.
///
/// Root cause. An open / increase commitment locks the opening bounty and the order margin through
///   `MarginClearinghouse.lockReservedSettlement` and `MarginClearinghouse.reserveCommittedOrderMargin`. Both call
///   `_checkpointCarryBeforeMarginChange`, which makes the Engine collect ALL carry accrued on the account's existing
///   position (`CfdEngine._realizeCarryFromSettlement`). `OrderReservationAccounting._reserveExecutionBounty` returns
///   commitment effects with `carryCollectedAssets == 0` for an open, `OrderLifecycleBook.recordCommitment` therefore
///   checks nothing, and `CfdOrderPolicyEvaluator.assessCommittedOrder` calls `_includeCommitment` only for closes.
///   The same carry is counted (a) by the close path at commit and at execution, and (b) by the evaluator itself for
///   the very same increase while the carry is still pending.
///
/// Production entrypoints only (Router V3 commitOrder / executeOrder / updateMarkPrice, Clearinghouse deposit, the
/// policy evaluator's public views). Release risk parameters. Constant basket price 1.0000 throughout.
contract OpenCommitCarryOutsideBoundsTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);
    address internal constant CAROL = address(0xCA201);
    address internal constant DAVE = address(0xDA7E);
    address internal constant KEEPER = address(0xBEE9E2);

    uint256 internal constant P0 = 100_000_000; // 1.0000
    uint256 internal constant DEPOSIT = 30_000e6;
    uint256 internal constant POSITION_SIZE = 400_000e18; // 400,000 USDG notional at 1.0000
    uint256 internal constant POSITION_MARGIN = 10_000e6;
    uint256 internal constant STEP_SIZE = 50_000e18; // 50,000 USDG notional increase / partial close
    uint256 internal constant STEP_MARGIN = 500e6;
    uint256 internal constant IDLE = 91 days; // Monday 2024-03-04 06:00 UTC -> Monday 2024-06-03 06:00 UTC

    uint256 internal clientNonce = 0xB0D001;

    struct Flow {
        uint256 balanceBeforeCommit;
        uint256 balanceAfterCommit;
        uint256 balanceAfterExecution;
        uint256 pledgeBeforeCommit;
        uint256 pledgeAfterCommit;
        uint256 poolCashGainAtCommit;
        uint256 assessedGross;
        uint256 assessedAction;
        uint256 receiptGross;
        uint256 receiptActionAssessed;
        uint256 receiptActionCollected;
        int256 receiptCarry;
        uint256 receiptCommitmentCarryCollected;
        uint256 receiptCommitmentSettlementBefore;
        uint256 receiptCommitmentSettlementAfter;
        uint8 status;
    }

    // ---- DEPLOYMENT.md "Initial release parameters" ----
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
        return 5_000_000e6;
    }

    function setUp() public override {
        super.setUp();
        _tickForward(1);
        router.updateMarkPrice(_live(P0));
    }

    // =================================================================================================================
    // 1. Four identical accounts hold the same LONG position, idle for 91 days at a constant price.
    //    ALICE : commits a small increase with tight debit ceilings -> Executed; the order debited more than both.
    //    BOB   : (control, close path) the same ceilings on a partial close -> commit reverts on the same carry.
    //    CAROL : (control, no pending carry) checkpoints carry first, same increase, same ceilings -> within bounds.
    //    DAVE  : (control, evaluator) the evaluator rejects the same increase with the same ceilings while the carry
    //            is still pending; once commit has collected it, the identical order is assessed as inside bounds.
    // =================================================================================================================
    function test_IncreaseExecutesAboveItsOwnDebitCeilings() public {
        address[4] memory traders = [ALICE, BOB, CAROL, DAVE];
        for (uint256 i; i < traders.length; ++i) {
            _fundTrader(traders[i], DEPOSIT);
            _openViaRouter(traders[i], CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN);
        }
        _warpTo(vm.getBlockTimestamp() + IDLE);
        router.updateMarkPrice(_live(P0));
        assertFalse(engine.isFadWindow(), "live market");
        assertFalse(engine.isOracleFrozen(), "live market");

        // ---- CAROL pays her accrued carry with an ordinary owner deposit of 1 USDG (no order involved). ----
        uint256 carolBefore = clearinghouse.balanceAssets(CAROL);
        _fundTrader(CAROL, 1e6);
        uint256 carolCarry = carolBefore + 1e6 - clearinghouse.balanceAssets(CAROL);
        console2.log("carry accrued per account over 91 idle days (USDG 6dp)      :", carolCarry);
        assertGt(carolCarry, 100e6, "material accrued carry");

        // ---- The ceilings: twice the evaluator's own quote for the increase on a carry-free twin. ----
        uint256 bounty = _quoteOpenOrderExecutionBountyAssets(STEP_SIZE);
        OrderV3Types.ExecutionAssessment memory quote =
            _assessHypotheticalIncrease(CAROL, _unbounded(), bounty);
        uint256 maxGross = 2 * quote.grossAccountDebitAssets;
        uint256 maxAction = 2 * quote.actionChargeAssessedAssets;
        console2.log("evaluator quote for the increase, gross debit (USDG 6dp)   :", quote.grossAccountDebitAssets);
        console2.log("evaluator quote for the increase, action charge (USDG 6dp) :", quote.actionChargeAssessedAssets);
        console2.log("trader ceiling maxGrossAccountDebitAssets (2x quote)       :", maxGross);
        console2.log("trader ceiling maxActionChargeAssets (2x quote)            :", maxAction);

        // ---- DAVE (evaluator control, before any commit): same order, same ceilings, carry still pending. ----
        {
            OrderV3Types.ExecutionBounds memory b = _unbounded();
            b.maxGrossAccountDebitAssets = maxGross;
            b.maxActionChargeAssets = maxAction;
            (bytes4 selector, uint256 kind, uint256 actual, uint256 limit) = _tryAssessIncrease(DAVE, b, bounty);
            assertEq(
                selector, ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector, "evaluator"
            );
            assertEq(kind, uint256(OrderV3Types.ConstraintKind.GrossAccountDebit), "gross debit is the violated bound");
            assertEq(limit, maxGross, "limit");
            console2.log("DAVE  pre-commit evaluator: ConstraintViolation(GrossAccountDebit) actual:", actual);
        }

        // ---- ALICE: the increase, tight ceilings. ----
        Flow memory alice = _increaseWithCeilings(ALICE, maxGross, maxAction);
        console2.log("--- ALICE: increase 50,000 with the ceilings above ---");
        _logFlow(alice);
        assertEq(alice.status, uint8(OrderV3Types.LifecycleStatus.Executed), "ALICE order executed");
        uint256 aliceCommitDebit = alice.balanceBeforeCommit - alice.balanceAfterCommit;
        uint256 aliceTotalDebit = alice.balanceBeforeCommit - alice.balanceAfterExecution;
        // The order ended Executed although the settlement it removed from the account exceeds BOTH ceilings.
        assertEq(alice.poolCashGainAtCommit, aliceCommitDebit, "the commit-time debit is carry paid to the pool");
        assertGt(aliceCommitDebit, maxGross, "commit alone debited more than maxGrossAccountDebitAssets");
        assertGt(aliceCommitDebit, maxAction, "commit alone charged more than maxActionChargeAssets");
        assertGt(aliceTotalDebit, maxGross, "order debited more than maxGrossAccountDebitAssets");
        // ...while the evaluator and the authenticated receipt report it as inside them.
        assertLe(alice.assessedGross, maxGross, "assessment says inside the ceiling");
        assertLe(alice.receiptGross, maxGross, "receipt says inside the ceiling");
        assertLe(alice.receiptActionAssessed, maxAction, "receipt says inside the ceiling");
        assertEq(alice.receiptGross + aliceCommitDebit, aliceTotalDebit, "unreported part == commit-time carry");
        assertEq(alice.receiptCommitmentCarryCollected, 0, "receipt commitment records no carry for the open");
        assertLt(uint256(alice.receiptCarry), 1e6, "receipt carry field omits the commit-time collection");
        console2.log("ALICE total settlement debit caused by the order           :", aliceTotalDebit);
        console2.log("ALICE excess over maxGrossAccountDebitAssets               :", aliceTotalDebit - maxGross);
        console2.log("ALICE debit / ceiling (x100)                               :", aliceTotalDebit * 100 / maxGross);

        // ---- BOB (close control): identical state, identical ceilings, partial close of the same size. ----
        {
            uint256 bobBefore = clearinghouse.balanceAssets(BOB);
            OrderV3Types.OrderRequest memory closeReq = _req(CfdTypes.Side.LONG, STEP_SIZE, 0, true);
            closeReq.bounds.maxGrossAccountDebitAssets = maxGross;
            closeReq.bounds.maxActionChargeAssets = maxAction;
            (bytes4 selector, uint256 kind, uint256 actual, uint256 limit) = _tryCommit(BOB, closeReq);
            assertEq(selector, IOrderLifecycleBook.OrderLifecycleBook__CommitmentBoundExceeded.selector, "close path");
            assertEq(kind, uint256(OrderV3Types.ConstraintKind.GrossAccountDebit), "gross debit");
            assertEq(limit, maxGross, "limit");
            assertEq(clearinghouse.balanceAssets(BOB), bobBefore, "rejected commit collected nothing");
            console2.log("--- BOB: partial close 50,000 with the same ceilings ---");
            console2.log("BOB   commit reverts CommitmentBoundExceeded(GrossAccountDebit) actual :", actual);
            console2.log("BOB   settlement debit                                     :", uint256(0));
        }

        // ---- CAROL (no-pending-carry control): same increase, same ceilings. ----
        Flow memory carol = _increaseWithCeilings(CAROL, maxGross, maxAction);
        console2.log("--- CAROL: same increase, same ceilings, carry checkpointed beforehand ---");
        _logFlow(carol);
        assertEq(carol.status, uint8(OrderV3Types.LifecycleStatus.Executed), "CAROL order executed");
        uint256 carolTotalDebit = carol.balanceBeforeCommit - carol.balanceAfterExecution;
        assertLe(carolTotalDebit, maxGross, "control stays inside the ceiling");
        assertLt(carol.balanceBeforeCommit - carol.balanceAfterCommit, 1e6, "control: only seconds of carry at commit");
        assertEq(
            carol.receiptGross + (carol.balanceBeforeCommit - carol.balanceAfterCommit),
            carolTotalDebit,
            "control: same identity, negligible commit-time part"
        );
        console2.log("CAROL total settlement debit caused by the order           :", carolTotalDebit);

        // ---- DAVE commits the order the evaluator rejected above: now it executes. ----
        Flow memory dave = _increaseWithCeilings(DAVE, maxGross, maxAction);
        assertEq(dave.status, uint8(OrderV3Types.LifecycleStatus.Executed), "DAVE order executed");
        console2.log("--- DAVE: the order the evaluator rejected pre-commit ---");
        console2.log("DAVE  total settlement debit caused by the order           :", dave.balanceBeforeCommit - dave.balanceAfterExecution);
        console2.log("DAVE  receipt grossAccountDebitAssets                      :", dave.receiptGross);

        // ---- Honest sizing: the excess is carry the account owed anyway. ----
        // CAROL paid the same carry through her deposit checkpoint; only the attribution to the order differs.
        console2.log("--- sizing: carry + increase cost paid since the idle period began ---");
        console2.log("ALICE (carry collected by the order's commit)              :", aliceTotalDebit);
        console2.log("CAROL (carry collected by her own deposit, then the order) :", carolCarry + carolTotalDebit);
    }

    // =================================================================================================================
    // 2. Consequence for a trader who derives the ceilings from the evaluator's quote on the live account state
    //    (quote x 1.25). DAVE has 91 days of pending carry, CAROL is the same account with the carry already paid.
    //    A large same-side open is queued ahead of both (FIFO), so the VPI they meet at execution is ~1.5x the quote.
    //    CAROL: the 25% tolerance works, her order fails with ConstraintViolation(ActionCharge).
    //    DAVE : his quote contained the carry, commit collected it outside the bounds, the ceilings keep a slack the
    //           size of the carry, and the order executes at a trade charge far above quote x 1.25.
    // =================================================================================================================
    function test_QuoteDerivedCeilingKeepsCarrySizedSlack() public {
        address whale = address(0x3A1E);
        _fundTrader(CAROL, DEPOSIT);
        _openViaRouter(CAROL, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN);
        _fundTrader(DAVE, DEPOSIT);
        _openViaRouter(DAVE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN);
        _fundTrader(whale, 100_000e6);
        _warpTo(vm.getBlockTimestamp() + IDLE);
        router.updateMarkPrice(_live(P0));
        _fundTrader(CAROL, 1e6); // CAROL's owner deposit collects her carry; DAVE's stays pending

        // A 600,000 same-side open is committed first and is pending at the FIFO head.
        uint64 whaleId = _commit(whale, _req(CfdTypes.Side.LONG, 600_000e18, 20_000e6, false));
        uint256 whaleCommit = vm.getBlockTimestamp();
        _tickForward(1);

        Quoted memory carol = _quoteAndCommitIncrease(CAROL);
        Quoted memory dave = _quoteAndCommitIncrease(DAVE);
        uint256 tradeQuote = carol.quotedAction; // carry-free twin: pure VPI + fee for the 50,000 increase
        console2.log("quoted trade charge for the increase (VPI + fee)           :", tradeQuote);
        console2.log("CAROL quote / ceiling (x1.25)                              :", carol.quotedAction, carol.maxAction);
        console2.log("DAVE  quote / ceiling (x1.25), quote includes pending carry:", dave.quotedAction, dave.maxAction);

        assertEq(
            uint8(_fillAt(whaleId, whaleCommit, P0).status), uint8(OrderV3Types.LifecycleStatus.Executed), "whale"
        );

        // CAROL: protected by her ceiling.
        OrderV3Types.ExecutionResult memory rc = _fillAt(carol.id, carol.commitTime, P0);
        assertEq(uint8(rc.status), uint8(OrderV3Types.LifecycleStatus.Failed), "CAROL order fails");
        assertEq(uint8(rc.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "by her ceiling");
        console2.log("CAROL order status (3 = Failed)                            :", uint256(rc.status));
        console2.log("CAROL terminal reason (8 = ConstraintViolation)            :", uint256(rc.terminalReason));

        // DAVE: same order, same market, executes.
        uint256 daveBefore = clearinghouse.balanceAssets(DAVE);
        (uint8 daveStatus, uint256 daveActionPaid) = _fillAndReadAction(dave.id, dave.commitTime);
        assertEq(daveStatus, uint8(OrderV3Types.LifecycleStatus.Executed), "DAVE order executes");
        console2.log("DAVE  order status (2 = Executed)                          :", uint256(daveStatus));
        console2.log("DAVE  action charge at execution (receipt)                 :", daveActionPaid);
        console2.log("DAVE  settlement debit at execution                        :", daveBefore - clearinghouse.balanceAssets(DAVE));
        console2.log("DAVE  charge / quoted trade charge (x100)                  :", daveActionPaid * 100 / tradeQuote);
        assertGt(daveActionPaid, tradeQuote * 125 / 100, "DAVE paid more than quote x 1.25 for the trade");
        assertLe(daveActionPaid, dave.maxAction, "inside the carry-inflated ceiling");
    }

    struct Quoted {
        uint64 id;
        uint256 commitTime;
        uint256 quotedAction;
        uint256 maxAction;
    }

    /// @dev Evaluator quote on the live account state, ceilings = quote x 1.25, commit.
    function _quoteAndCommitIncrease(
        address trader
    ) internal returns (Quoted memory q) {
        OrderV3Types.ExecutionAssessment memory a = _assessHypotheticalIncrease(
            trader, _unbounded(), _quoteOpenOrderExecutionBountyAssets(STEP_SIZE)
        );
        q.quotedAction = a.actionChargeAssessedAssets;
        OrderV3Types.OrderRequest memory req = _req(CfdTypes.Side.LONG, STEP_SIZE, STEP_MARGIN, false);
        req.bounds.maxGrossAccountDebitAssets = a.grossAccountDebitAssets * 125 / 100;
        req.bounds.maxActionChargeAssets = a.actionChargeAssessedAssets * 125 / 100;
        q.maxAction = req.bounds.maxActionChargeAssets;
        q.id = _commit(trader, req);
        q.commitTime = vm.getBlockTimestamp();
        _tickForward(1);
    }

    function _fillAndReadAction(
        uint64 id,
        uint256 commitTime
    ) internal returns (uint8 status, uint256 actionAssessed) {
        vm.recordLogs();
        OrderV3Types.ExecutionResult memory r = _fillAt(id, commitTime, P0);
        status = uint8(r.status);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector) {
                (,,, OrderV3Types.OrderReceipt memory receipt) =
                    abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                actionAssessed = receipt.economics.actionChargeAssessedAssets;
            }
        }
    }

    // ------------------------------------------------------------------------------------------------- helpers

    function _increaseWithCeilings(
        address trader,
        uint256 maxGross,
        uint256 maxAction
    ) internal returns (Flow memory f) {
        OrderV3Types.OrderRequest memory req = _req(CfdTypes.Side.LONG, STEP_SIZE, STEP_MARGIN, false);
        req.bounds.maxGrossAccountDebitAssets = maxGross;
        req.bounds.maxActionChargeAssets = maxAction;

        f.balanceBeforeCommit = clearinghouse.balanceAssets(trader);
        f.pledgeBeforeCommit = _pledge(trader);
        uint256 poolCashBefore = settlementToken.balanceOf(address(pool));
        uint64 id = _commit(trader, req);
        f.poolCashGainAtCommit = settlementToken.balanceOf(address(pool)) - poolCashBefore;
        uint256 commitTime = vm.getBlockTimestamp();
        f.balanceAfterCommit = clearinghouse.balanceAssets(trader);
        f.pledgeAfterCommit = _pledge(trader);

        _tickForward(1);
        OrderV3Types.ExecutionAssessment memory a =
            policyEvaluator.assessCommittedOrder(address(engine), id, KEEPER, P0);
        f.assessedGross = a.grossAccountDebitAssets;
        f.assessedAction = a.actionChargeAssessedAssets;

        vm.recordLogs();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, _ticks(P0, commitTime + 1, commitTime));
        f.status = uint8(r.status);
        f.balanceAfterExecution = clearinghouse.balanceAssets(trader);

        Vm.Log[] memory logs = vm.getRecordedLogs();
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector) {
                (,,, OrderV3Types.OrderReceipt memory receipt) =
                    abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                f.receiptGross = receipt.economics.grossAccountDebitAssets;
                f.receiptActionAssessed = receipt.economics.actionChargeAssessedAssets;
                f.receiptActionCollected = receipt.economics.actionChargeCollectedAssets;
                f.receiptCarry = receipt.economics.carryAssets;
                f.receiptCommitmentCarryCollected = receipt.commitment.carryCollectedAssets;
                f.receiptCommitmentSettlementBefore = receipt.commitment.settlementBeforeAssets;
                f.receiptCommitmentSettlementAfter = receipt.commitment.settlementAfterAssets;
            }
        }
    }

    function _logFlow(
        Flow memory f
    ) internal pure {
        console2.log("settlement balance before commit                           :", f.balanceBeforeCommit);
        console2.log("settlement debit at COMMIT (carry, outside every bound)    :", f.balanceBeforeCommit - f.balanceAfterCommit);
        console2.log("  of which taken from position pledge                      :", f.pledgeBeforeCommit - f.pledgeAfterCommit);
        console2.log("  pool cash received at COMMIT (carry booked to LPs)       :", f.poolCashGainAtCommit);
        console2.log("settlement debit at EXECUTION (trade cost + bounty)        :", f.balanceAfterCommit - f.balanceAfterExecution);
        console2.log("evaluator grossAccountDebitAssets at execution             :", f.assessedGross);
        console2.log("evaluator actionChargeAssessedAssets at execution          :", f.assessedAction);
        console2.log("receipt   grossAccountDebitAssets                          :", f.receiptGross);
        console2.log("receipt   actionChargeAssessedAssets                       :", f.receiptActionAssessed);
        console2.log("receipt   actionChargeCollectedAssets                      :", f.receiptActionCollected);
        console2.log("receipt   carryAssets                                      :", f.receiptCarry);
        console2.log("receipt   commitment.carryCollectedAssets                  :", f.receiptCommitmentCarryCollected);
        console2.log("receipt   commitment.settlementBefore / After              :", f.receiptCommitmentSettlementBefore, f.receiptCommitmentSettlementAfter);
        console2.log("order lifecycle status (2 = Executed)                      :", uint256(f.status));
    }

    function _hypotheticalIncrease(
        address trader
    ) internal view returns (CfdTypes.Order memory order) {
        order = CfdTypes.Order({
            account: trader,
            sizeDelta: STEP_SIZE,
            marginDelta: STEP_MARGIN,
            targetPrice: 1,
            commitTime: uint64(vm.getBlockTimestamp()),
            commitBlock: uint64(vm.getBlockNumber()),
            orderId: 0,
            side: CfdTypes.Side.LONG,
            isClose: false
        });
    }

    function _assessHypotheticalIncrease(
        address trader,
        OrderV3Types.ExecutionBounds memory bounds,
        uint256 bounty
    ) internal view returns (OrderV3Types.ExecutionAssessment memory) {
        return policyEvaluator.assessOrder(
            address(engine), _hypotheticalIncrease(trader), KEEPER, P0, pool.totalAssets(), bounds, bounty
        );
    }

    function _tryAssessIncrease(
        address trader,
        OrderV3Types.ExecutionBounds memory bounds,
        uint256 bounty
    ) internal view returns (bytes4 selector, uint256 kind, uint256 actual, uint256 limit) {
        try policyEvaluator.assessOrder(
            address(engine), _hypotheticalIncrease(trader), KEEPER, P0, pool.totalAssets(), bounds, bounty
        ) returns (
            OrderV3Types.ExecutionAssessment memory
        ) {
            return (bytes4(0), 0, 0, 0);
        } catch (bytes memory err) {
            return _decodeConstraint(err);
        }
    }

    function _tryCommit(
        address trader,
        OrderV3Types.OrderRequest memory req
    ) internal returns (bytes4 selector, uint256 kind, uint256 actual, uint256 limit) {
        vm.prank(trader);
        try router.commitOrder(req) returns (uint64) {
            return (bytes4(0), 0, 0, 0);
        } catch (bytes memory err) {
            return _decodeConstraint(err);
        }
    }

    function _decodeConstraint(
        bytes memory err
    ) internal pure returns (bytes4 selector, uint256 kind, uint256 actual, uint256 limit) {
        if (err.length < 100) {
            if (err.length >= 4) {
                assembly ("memory-safe") {
                    selector := mload(add(err, 0x20))
                }
            }
            return (selector, 0, 0, 0);
        }
        assembly ("memory-safe") {
            selector := mload(add(err, 0x20))
            kind := mload(add(err, 0x24))
            actual := mload(add(err, 0x44))
            limit := mload(add(err, 0x64))
        }
    }

    function _pledge(
        address account
    ) internal view returns (uint256 margin) {
        (, margin,,,,,) = engine.positions(account);
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _tickForward(
        uint256 secs
    ) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + secs);
    }

    function _warpTo(
        uint256 timestamp
    ) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(timestamp);
    }

    /// @dev One signed tick per basket feed: (price, publishTime, prevPublishTime).
    function _ticks(
        uint256 price,
        uint256 publishTime,
        uint256 prevPublishTime
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, prevPublishTime));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _live(
        uint256 price
    ) internal view returns (bytes[] memory) {
        uint256 nowTs = vm.getBlockTimestamp();
        return _ticks(price, nowTs, nowTs - 1);
    }

    function _unbounded() internal view returns (OrderV3Types.ExecutionBounds memory) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        return OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp()) + uint64(window),
            executionWindowSeconds: window,
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

    function _req(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory) {
        return OrderV3Types.OrderRequest({
            clientOrderId: bytes32(clientNonce++),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: ((side == CfdTypes.Side.LONG) == isClose) ? type(uint256).max : 1,
            isClose: isClose,
            bounds: _unbounded()
        });
    }

    function _commit(
        address trader,
        OrderV3Types.OrderRequest memory req
    ) internal returns (uint64 id) {
        vm.prank(trader);
        id = router.commitOrder(req);
    }

    /// @dev Keeper executes `id` with the unique first tick after its commit.
    function _fillAt(
        uint64 id,
        uint256 commitTime,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory r) {
        if (vm.getBlockTimestamp() <= commitTime) {
            _tickForward(commitTime + 1 - vm.getBlockTimestamp());
        } else {
            vm.roll(vm.getBlockNumber() + 1);
        }
        vm.prank(KEEPER);
        r = router.executeOrder(id, _ticks(price, commitTime + 1, commitTime));
    }

    function _openViaRouter(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        uint256 before = _size(trader);
        uint64 id = _commit(trader, _req(side, size, margin, false));
        OrderV3Types.ExecutionResult memory r = _fillAt(id, vm.getBlockTimestamp(), P0);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open fills");
        assertEq(_size(trader), before + size, "position opened");
    }

}
```

## Recorded Test Output

The output below is from the successful isolated test run on the audited commit:

```text
Compiling 194 files with Solc 0.8.35
Compiler run successful with warnings:

Ran 2 tests for test/perps/bounds_only/OpenCommitCarryOutsideBounds.t.sol:OpenCommitCarryOutsideBoundsTest
[PASS] test_IncreaseExecutesAboveItsOwnDebitCeilings() (gas: 16127669)
Logs:
  carry accrued per account over 91 idle days (USDG 6dp)      : 1525048004
  evaluator quote for the increase, gross debit (USDG 6dp)   : 182502524
  evaluator quote for the increase, action charge (USDG 6dp) : 182302524
  trader ceiling maxGrossAccountDebitAssets (2x quote)       : 365005048
  trader ceiling maxActionChargeAssets (2x quote)            : 364605048
  DAVE  pre-commit evaluator: ConstraintViolation(GrossAccountDebit) actual: 1708746532
  --- ALICE: increase 50,000 with the ceilings above ---
  settlement balance before commit                           : 29679863975
  settlement debit at COMMIT (carry, outside every bound)    : 1522555596
    of which taken from position pledge                      : 1522555596
    pool cash received at COMMIT (carry booked to LPs)       : 1522555596
  settlement debit at EXECUTION (trade cost + bounty)        : 182453370
  evaluator grossAccountDebitAssets at execution             : 182453370
  evaluator actionChargeAssessedAssets at execution          : 182253370
  receipt   grossAccountDebitAssets                          : 182453370
  receipt   actionChargeAssessedAssets                       : 182253370
  receipt   actionChargeCollectedAssets                      : 182253370
  receipt   carryAssets                                      : 194
  receipt   commitment.carryCollectedAssets                  : 0
  receipt   commitment.settlementBefore / After              : 0 0
  order lifecycle status (2 = Executed)                      : 2
  ALICE total settlement debit caused by the order           : 1705008966
  ALICE excess over maxGrossAccountDebitAssets               : 1340003918
  ALICE debit / ceiling (x100)                               : 467
  --- BOB: partial close 50,000 with the same ceilings ---
  BOB   commit reverts CommitmentBoundExceeded(GrossAccountDebit) actual : 1523802157
  BOB   settlement debit                                     : 0
  --- CAROL: same increase, same ceilings, carry checkpointed beforehand ---
  settlement balance before commit                           : 27516174130
  settlement debit at COMMIT (carry, outside every bound)    : 194
    of which taken from position pledge                      : 194
    pool cash received at COMMIT (carry booked to LPs)       : 194
  settlement debit at EXECUTION (trade cost + bounty)        : 187440363
  evaluator grossAccountDebitAssets at execution             : 187440363
  evaluator actionChargeAssessedAssets                       : 187240363
  receipt   grossAccountDebitAssets                          : 187440363
  receipt   actionChargeAssessedAssets                       : 187240363
  receipt   actionChargeCollectedAssets                      : 187240363
  receipt   carryAssets                                      : 201
  receipt   commitment.carryCollectedAssets                  : 0
  receipt   commitment.settlementBefore / After              : 0 0
  order lifecycle status (2 = Executed)                      : 2
  CAROL total settlement debit caused by the order           : 187440557
  --- DAVE: the order the evaluator rejected pre-commit ---
  DAVE  total settlement debit caused by the order           : 1718668259
  DAVE  receipt grossAccountDebitAssets                      : 192374387
  --- sizing: carry + increase cost paid since the idle period began ---
  ALICE (carry collected by the order's commit)              : 1705008966
  CAROL (carry collected by her own deposit, then the order) : 1712488561

[PASS] test_QuoteDerivedCeilingKeepsCarrySizedSlack() (gas: 10597960)
Logs:
  quoted trade charge for the increase (VPI + fee)           : 102444026
  CAROL quote / ceiling (x1.25)                              : 102444026 128055032
  DAVE  quote / ceiling (x1.25), quote includes pending carry: 863844979 1079806223
  CAROL order status (3 = Failed)                            : 3
  CAROL terminal reason (8 = ConstraintViolation)            : 8
  DAVE  order status (2 = Executed)                          : 2
  DAVE  action charge at execution (receipt)                 : 162344064
  DAVE  settlement debit at execution                        : 162544064
  DAVE  charge / quoted trade charge (x100)                  : 158

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 154.72ms (73.23ms CPU time)
Ran 1 test suite in 177.59ms (154.72ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
========== EXIT STATUS ==========
0
========== PRODUCTION CONTRACT DIFF ==========
```

The empty production diff confirms the result was obtained without changes to the audited `packages/perps/src` contracts.

