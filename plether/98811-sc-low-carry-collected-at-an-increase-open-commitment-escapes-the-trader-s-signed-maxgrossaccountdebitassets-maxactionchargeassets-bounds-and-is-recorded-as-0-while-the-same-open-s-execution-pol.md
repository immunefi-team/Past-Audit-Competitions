# \#98811 [SC-Low] Carry collected at an increase-open commitment escapes the trader's signed `maxGrossAccountDebitAssets` / `maxActionChargeAssets` bounds and is recorded as 0, while the same open's execution policy counts carry inside those bounds

**Submitted on Nov 10th 2026 at 10:38:02 UTC by @RUNSUN for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98811
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

**Target**: `packages/perps/src` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (`router/OrderReservationAccounting.sol`, `MarginClearinghouse.sol`, `CfdEngine.sol`, `OrderLifecycleBook.sol`, `CfdOrderPolicyEvaluator.sol`, `CfdOrderPolicyEvaluatorBase.sol`)

## Brief/Intro

**The protocol's own open-order policy treats carry as part of what a trader's signed bounds limit.** When an open executes, `CfdOrderPolicyEvaluatorBase.sol:346-348` puts the realised carry inside the gross account debit and the action charge that are enforced against the trader's `maxGrossAccountDebitAssets` and `maxActionChargeAssets`:

```solidity
assessment.grossAccountDebitAssets = realizedCarryAssets + positiveTradeCostAssets + executionBountyAssets;   // :346
assessment.actionChargeAssessedAssets = _netOpenActionCharge(realizedCarryAssets, delta.tradeCostAssets);   // :347
assessment.actionChargeCollectedAssets = realizedCarryAssets + positiveTradeCostAssets;                     // :348
```

**But for an open that increases an existing position, the carry is taken one step earlier, at commitment, where nothing bounds it:**
- the commit's reservation locks call the clearinghouse carry hook, which collects all accrued carry (`router/OrderReservationAccounting.sol:127` -> `MarginClearinghouse.sol:822` -> `CfdEngine.sol:548`);
- the open path hands the Book commitment effects with carry hard-coded to 0, so the Book checks and stores 0;
- at execution, the carry is no longer pending, so the policy at :346-348 never sees it; and the commitment carry is re-added to the bounds only for closes (`CfdOrderPolicyEvaluator.sol:65-67`).

So an increase-open signed with 0.60 USDG maxima debits 140 USDG at commitment, executes, and produces an authenticated `OrderFinalized` receipt that says the order stayed within its limits.

## Vulnerability Details

**1. The open path returns commitment effects with carry 0** (`router/OrderReservationAccounting.sol:108-128`):

```solidity
function _reserveExecutionBounty(address account, uint256 sizeDelta, uint256 bounty, bool isClose)
    internal returns (CfdEnginePlanTypes.CloseCommitment memory effects)
{
    if (isClose) {
        if (bounty != 0) { revert ICfdEngineTypes.CfdEngine__InvalidRiskParams(); }
        return _reserveCloseExecutionBounty(account, sizeDelta);   // close: carry projected, bounded, recorded
    }
    effects.bountyFromFreeAssets = bounty;                         // open: carryCollectedAssets stays 0
    if (bounty == 0) { return effects; }
    if (clearinghouse.getAccountAssetBuckets(account).freeSettlementAssets < bounty) {
        revert OrderRouter__InsufficientFreeEquity();
    }
    clearinghouse.lockReservedSettlement(account, bounty);         // :127 - collects all accrued carry
}
```

**2. The locks collect all accrued carry** (`MarginClearinghouse.sol:818-823`; the same hook runs in `reserveCommittedOrderMargin` at `:632`, reached from `router/OrderReservationAccounting.sol:145`; `CfdEngine.sol:535-549`):

```solidity
function lockReservedSettlement(address account, uint256 amountAssets) external onlyReservedSettlementLocker {
    _checkpointCarryBeforeMarginChange(account);   // -> engine.realizeCarryBeforeMarginChange(account)
    _lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
}

function realizeCarryBeforeMarginChange(address account) external nonReentrant {
    if (msg.sender != address(clearinghouse)) { revert CfdEngine__NotClearinghouse(); }
    StoredPosition storage pos = _positions[account];
    if (pos.lots == 0) { return; }
    bytes32 expectedOldHash = _beginTerminalCurveMutation(account);
    _realizeCarryFromSettlement(account, pos);     // collects ALL accrued carry, pledge first, then free settlement
    _endTerminalCurveMutation(account, expectedOldHash);
}
```

**3. The Book bounds and records the 0** (`router/OrderCommitHandler.sol:21-35`: `_reserveExecutionBounty` at `:28`, then `_onOrderCommitted` at `:32` -> `OrderLifecycleBook.recordCommitment`, `OrderLifecycleBook.sol:335-360`):

```solidity
uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;   // 0 + bounty
if (gross > pending.bounds.maxGrossAccountDebitAssets) { revert ...CommitmentBoundExceeded(GrossAccountDebit, ...); }
if (effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets) { revert ...CommitmentBoundExceeded(ActionCharge, ...); }
...
pending.commitment = StoredCommitment({ carryCollectedAssets: effects.carryCollectedAssets, ... });   // stores 0
```

**4. Execution re-adds commitment carry to the bounds only for closes** (`CfdOrderPolicyEvaluator.sol:65-67`, `CfdOrderPolicyEvaluatorBase.sol:474-486`):

```solidity
assessment = _assessSnapshot(context, order, pending.bounds, planner, snapshot);
if (order.isClose) {
    _includeCommitment(assessment, pending.commitment, pending.bounds, pending.executionBountyAssets);   // adds commitment carry, re-enforces bounds
}
```

### The whole class: every carry-collection path against the signed bounds

All carry is collected by one primitive, `CfdEngine._realizeCarryFromSettlement` (`CfdEngine.sol:1407`). Every path that reaches it, with whether a signed order's bounds apply:

| Path | Collection site | Inside a signed order? | Carry checked against `maxGrossAccountDebitAssets` / `maxActionChargeAssets`? | Book / receipt records it? |
|---|---|---|---|---|
| **Increase-open commit, bounty lock** (`OrderReservationAccounting.sol:127` -> `MarginClearinghouse.sol:822`) | `CfdEngine.sol:548` | yes | **no** (effects carry = 0; not re-added at execution) | **no** |
| **Increase-open commit, margin reservation** (`OrderReservationAccounting.sol:145` -> `MarginClearinghouse.sol:632`) | `CfdEngine.sol:548` | yes | **no** (same transaction; collects whatever the first lock left) | **no** |
| Close commit (`reserveCloseOrderExecutionBounty`, `CfdEngine.sol:598` -> settlement sidecar `:343`) | `CfdEngine.sol:1162` | yes | yes (Book `:335-347`; re-enforced at execution via `_includeCommitment`) | yes |
| Open execution, increase (`_applyOpen`) | `CfdEngine.sol:1293` | yes | yes, but only for carry accrued after the commit (`:346-348`) | yes, that part only |
| Close execution, ordinary and protection attempts (`_applyClose`) | `CfdEngine.sol:1313` | yes | yes (`CfdOrderPolicyEvaluatorBase.sol:410-414`, enforced at `:189`) | yes |
| Deposit / withdraw / `addMargin` / `settleTraderClaim` / protection create and cancel | `CfdEngine.sol:548`, `:1162`, `:518`, `:569` | no (account actions, documented) | n/a | n/a |
| Liquidation; bounty credits to keepers and executors | `CfdEngine.sol:1330`, `:215` | no (forced settlement / beneficiary side) | n/a | n/a |
| Fresh open (flat account), `depositFor` | none | - | - | - |

**The class is exactly one code path:** the two reservation hooks of an increase-open commitment. Every other path that runs inside a signed order bounds and records its carry. The asymmetry has two halves:
- the close commit projects, bounds and records carry, while the open commit hard-codes 0;
- execution re-adds commitment carry to the bounds for closes, but skips it for opens.

## Why this is not by design

- **The open execution policy bounds carry** (`CfdOrderPolicyEvaluatorBase.sol:346-348` above). If carry were never meant to count against an open's signed limits, that policy would not put it in the gross debit and the action charge. The commit-time collection removes exactly the amount that policy would otherwise bound.
- **The patched test in the PoC shows the intended behaviour.** With the commit-time collection disabled, the carry reaches that policy at execution, and the same 0.60 USDG signed limit fires: terminal `ConstraintViolation`, total debit 0.10 USDG (the bounty only).
- **The documented guarantee is general, not close-only.** `SECURITY.md:368` lists, as a Router invariant: "Bounded authority | Every fresh order pins a nonzero target, deadline, mode mask, config hash, and inclusive financial limits; zero maxima are real zero allowances rather than unbounded sentinels".
- `ACCOUNTING_SPEC.md:943-945` describes the close commitment ("Collected commitment carry is included exactly once in gross account debit, assessed/collected action charges, and carry totals for the user's financial bounds"). The open commitment collects carry just the same, but has no such accounting at all. That is the gap this report describes, not a documented exception for opens.
- Collecting carry on margin changes is documented design and is not disputed here. What is reported is that, inside a signed open order, the collected amount bypasses the order's own limits and the receipt.

## Impact Details

- **The signed limits do not bind.** In the PoC the trader signs `maxGrossAccountDebitAssets = maxActionChargeAssets = 0.60 USDG` (bounty 0.10 + fee 0.40 + 0.10 slack).
  - The commit alone debits **139.92 USDG**; the order executes; the order's total account debit is **140.42 USDG, 234x the signed gross-debit limit**.
  - The overrun equals the account's accrued carry, so it grows with position size, pool utilisation and time since the last checkpoint, with no cap. With `maxGrossAccountDebitAssets = 0` meant as "a real zero allowance", any accrued carry breaks it.
- **The authenticated receipt is wrong.** The `OrderFinalized` receipt reports `grossAccountDebitAssets = 0.500115` and `commitment.carryCollectedAssets = 0`, i.e. within the limits.
  - Integrators, automated agents and off-chain accounting that rely on the signed maxima as hard caps, or on the receipt as the record of what an order cost, get both wrong.
- **What is not claimed.** No value is lost: the carry is genuinely owed by the account and would otherwise be collected on its next margin change, close or liquidation, in the same order (pledge first, then free settlement). The report is about the broken bound and the false receipt, which is why it is filed as Low under "Contract fails to deliver promised returns, but doesn't lose value".

## Known-issues check

- **Not in the committed audits** (`audits/`):
  - one prior audit report's carry items concern avoiding carry through rounding by checkpointing early (MED-22), assessed-versus-collected carry on the close side (LOW-34), and a preview that can overstate committable margin (LOW-53);
  - the protection-close "no effective financial caps" item concerns protection closes with unbounded maxima;
  - none of them concerns carry collected at an open commitment outside the signed limits.
  - The Cantina, collaborative and pre-audit reports have no hit for this path.
- **Not one of the programme's listed known issues.**

## Recommendation

Close both halves of the asymmetry:
1. **Bound and record the carry an open commitment collects**, the way the close commitment does: realise the account's accrued carry explicitly at the start of the open commitment and report it in the commitment effects, so `recordCommitment` checks it against `maxGrossAccountDebitAssets` / `maxActionChargeAssets` and stores it for the receipt. The two reservation hooks then find nothing left to collect.
2. **Re-include commitment carry at execution for opens as well**: remove the `order.isClose` gate around `_includeCommitment` at `CfdOrderPolicyEvaluator.sol:65-67`, so a recorded open commitment carry is re-bounded with the execution assessment.

Fix sketch for (1) (not compiled; `realizeCarryForCommit` stands for a router-callable variant of `realizeCarryBeforeMarginChange`, which today accepts calls from the clearinghouse only):

```diff
     effects.bountyFromFreeAssets = bounty;
+    uint256 settlementBefore = clearinghouse.balanceAssets(account);
+    engine.realizeCarryForCommit(account);          // collect accrued carry inside the bounded commit
+    uint256 settlementAfter = clearinghouse.balanceAssets(account);
+    effects.settlementBeforeAssets = settlementBefore;
+    effects.settlementAfterAssets = settlementAfter;
+    effects.carryCollectedAssets = settlementBefore - settlementAfter;
     if (bounty == 0) {
         return effects;
     }
```

```diff
-    if (order.isClose) {
-        _includeCommitment(assessment, pending.commitment, pending.bounds, pending.executionBountyAssets);
-    }
+    _includeCommitment(assessment, pending.commitment, pending.bounds, pending.executionBountyAssets);
```

The alternative, which the PoC's patched test emulates, is to skip the carry checkpoint in the two reservation hooks during an open commitment and let execution collect the carry under the policy at `:346-348`.

## Limits of this report

- The fix is a sketch; the patched PoC test emulates the fix with `vm.mockCall` on `realizeCarryBeforeMarginChange` during the commit, not with a source change.
- The carry amount in the PoC depends on the fixture's utilisation-scaled carry (300k notional, 14 days). A smaller or younger position gives a proportionally smaller overrun; the bound is broken for any accrued carry above the slack the trader signs.
- The class table was built by reading the code at the pinned commit (every collection site traced to `CfdEngine._realizeCarryFromSettlement`); only the increase-open path is executed in the PoC.
- Not measured: how integrators actually consume the receipt fields.


## Proof of Concept

## Proof of Concept

**File**: `packages/perps/test/perps/poc/IncreaseOpenCarryBounds.t.sol` (uses the repository's own `BasePerpTest` fixture; no source change).

**Run** (from `packages/perps`):
```
forge test --match-path test/perps/poc/IncreaseOpenCarryBounds.t.sol -vv
```

**Three tests:**
- `test_vulnerable_IncreaseOpenCarryExceedsGrossDebitAndActionChargeBounds`, on the unmodified code:
  - a 300,000 USDG LONG aged 14 days, then a 1,000 USDG increase signed with 0.60 USDG maxima;
  - asserts the commit debits more than the limit while the Book records 0;
  - asserts the order executes and the total debit and action charge exceed the signed limits;
  - asserts the authenticated receipt reports `carryCollectedAssets = 0` and a gross debit within the limit.
- `test_patched_DeferredCarryIsBoundedAndNoExcessDebit`, the fix emulated:
  - the carry hook is a no-op during the open commit only, so the carry reaches the evaluator at execution;
  - the same signed limit then fires: terminal `ConstraintViolation`, total debit 0.10 USDG (the bounty);
  - the over-limit debit does not occur.
- `test_sibling_CloseCommitWithSameBoundReverts`, the guarded sibling: the same aged state and the same 0.60 limit on a close commitment reverts `OrderLifecycleBook__CommitmentBoundExceeded`, and nothing is debited.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineCore} from "@plether/perps/interfaces/ICfdEngineCore.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Shared V3 request builder.
abstract contract PoCBase is BasePerpTest {

    function _v3Request(
        bytes32 tag,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = keccak256(abi.encode(tag, router.nextCommitId()));
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        // Always-satisfied slippage boundary (see OrderValidationLib.checkSlippage).
        r.targetPrice = ((side == CfdTypes.Side.LONG) == isClose) ? type(uint256).max : 1;
        r.isClose = isClose;
        r.bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        r.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        r.bounds.allowedExecutionModes = 7;
        r.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        r.bounds.maxExecutionBountyAssets = type(uint256).max;
        r.bounds.maxExecutionNotionalAssets = type(uint256).max;
        r.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        r.bounds.maxActionChargeAssets = type(uint256).max;
        r.bounds.maxExplicitFeesAssets = type(uint256).max;
        r.bounds.maxPostPositionSize = type(uint256).max;
        r.bounds.maxPostLeverageBps = type(uint32).max;
    }

    function _finalizedReceipt(
        Vm.Log[] memory logs,
        uint64 id
    ) internal view returns (bool found, OrderV3Types.OrderReceipt memory receipt) {
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter == address(router.lifecycleBook()) && logs[i].topics.length == 4
                    && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector
                    && uint64(uint256(logs[i].topics[1])) == id
            ) {
                (,,, receipt) = abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                return (true, receipt);
            }
        }
    }

}

/// ============================================================================================
/// Increase-open commitment collects accrued carry that the Book records as 0, so the trader's
/// maxGrossAccountDebitAssets / maxActionChargeAssets bounds never see it.
/// ============================================================================================
contract IncreaseOpenCarryBoundsTest is PoCBase {

    address internal constant TRADER = address(0xA11CE);
    address internal constant KEEPER = address(0xB0B);
    uint256 internal constant PRICE = 1e8;
    uint256 internal constant BASE_SIZE = 300_000e18; // 300,000 USDG notional (utilisation-scaled carry)
    uint256 internal constant INC_SIZE = 1_000e18; // 1,000 USDG notional increase
    uint256 internal constant INC_MARGIN = 200e6;
    uint256 internal constant AGE = 14 days; // multiple of 7 days: lands on the same weekday (no FAD)

    struct Ctx {
        uint256 bounty;
        uint256 fee;
        uint256 bound;
        uint256 pendingCarry;
        uint256 custodyBefore;
    }

    function _agedPosition() internal returns (Ctx memory c) {
        _fundTrader(TRADER, 40_000e6);
        vm.prank(TRADER);
        uint64 id = router.commitOrder(CfdTypes.Side.LONG, BASE_SIZE, 30_000e6, 0, false);
        bytes[] memory update = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        router.executeOrder(id, update);
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, BASE_SIZE, "base position open");

        vm.warp(vm.getBlockTimestamp() + AGE);
        vm.roll(vm.getBlockNumber() + 1);

        c.bounty = _quoteOpenOrderExecutionBountyAssets(INC_SIZE);
        c.fee = _engineExecutionFeeAssets(INC_SIZE, PRICE);
        // The trader authorises at most bounty + fee + 0.10 USDG of gross account debit / action charge.
        c.bound = c.bounty + c.fee + 100_000;
        c.pendingCarry = engine.unsettledCarryAssets(TRADER) + _expectedIndexedCarryAssets(TRADER);
        c.custodyBefore = clearinghouse.balanceAssets(TRADER);
        console2.log("pending carry (USDG atoms)", c.pendingCarry);
        console2.log("bounty", c.bounty);
        console2.log("fee", c.fee);
        console2.log("trader bound maxGross/maxActionCharge", c.bound);
        assertGt(c.pendingCarry, c.bound, "setup: accrued carry exceeds the trader's bound");
    }

    function _boundedIncrease(
        Ctx memory c
    ) internal view returns (OrderV3Types.OrderRequest memory r) {
        r = _v3Request("inc", CfdTypes.Side.LONG, INC_SIZE, INC_MARGIN, false);
        r.bounds.maxGrossAccountDebitAssets = c.bound;
        r.bounds.maxActionChargeAssets = c.bound;
    }

    /// VULNERABLE ARM (unmodified a82d4fef): the bound is silently exceeded.
    function test_vulnerable_IncreaseOpenCarryExceedsGrossDebitAndActionChargeBounds() public {
        Ctx memory c = _agedPosition();
        OrderV3Types.OrderRequest memory r = _boundedIncrease(c);

        vm.prank(TRADER);
        uint64 id = router.commitOrder(r);
        uint256 carryAtCommit = c.custodyBefore - clearinghouse.balanceAssets(TRADER);
        uint256 recordedCarry = router.lifecycleBook().pendingIntent(id).commitment.carryCollectedAssets;
        console2.log("carry debited at commit", carryAtCommit);
        console2.log("carry recorded by Book", recordedCarry);
        assertEq(recordedCarry, 0, "Book records zero commitment carry");
        assertGt(carryAtCommit, c.bound, "commit alone debits more than maxGrossAccountDebitAssets");

        bytes[] memory update = _mockPythUpdateData(PRICE);
        vm.recordLogs();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(id, update);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executes");

        uint256 totalDebit = c.custodyBefore - clearinghouse.balanceAssets(TRADER);
        console2.log("total account debit for the order", totalDebit);
        // HARM: the order's gross account debit exceeds the trader-signed maximum.
        assertGt(totalDebit, r.bounds.maxGrossAccountDebitAssets, "gross debit above signed bound");
        // HARM: action charge collected (carry + fee) exceeds the signed maximum.
        assertGt(carryAtCommit + c.fee, r.bounds.maxActionChargeAssets, "action charge above signed bound");

        (bool found, OrderV3Types.OrderReceipt memory receipt) = _finalizedReceipt(vm.getRecordedLogs(), id);
        assertTrue(found, "receipt");
        console2.log("receipt.grossAccountDebitAssets", receipt.economics.grossAccountDebitAssets);
        console2.log("receipt.commitment.carryCollectedAssets", receipt.commitment.carryCollectedAssets);
        assertEq(receipt.commitment.carryCollectedAssets, 0, "authenticated receipt omits the carry");
        assertLe(receipt.economics.grossAccountDebitAssets, r.bounds.maxGrossAccountDebitAssets, "receipt claims compliance");
    }

    /// PATCHED ARM: emulates the minimal fix "do not collect carry outside the bounded path on open commitment"
    /// by making the clearinghouse carry hook a no-op during commit only. The carry then reaches the evaluator at
    /// execution (CfdOrderPolicyEvaluatorBase:346) and the same signed bound fires; no excess debit occurs.
    function test_patched_DeferredCarryIsBoundedAndNoExcessDebit() public {
        Ctx memory c = _agedPosition();
        OrderV3Types.OrderRequest memory r = _boundedIncrease(c);

        vm.mockCall(
            address(engine), abi.encodeWithSelector(ICfdEngineCore.realizeCarryBeforeMarginChange.selector, TRADER), ""
        );
        vm.prank(TRADER);
        uint64 id = router.commitOrder(r);
        vm.clearMockedCalls();
        assertEq(c.custodyBefore, clearinghouse.balanceAssets(TRADER), "patched: no unbounded debit at commit");

        bytes[] memory update = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(id, update);
        console2.log("patched terminal reason", uint8(res.terminalReason));
        assertEq(uint8(res.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "bound enforced");
        uint256 totalDebit = c.custodyBefore - clearinghouse.balanceAssets(TRADER);
        console2.log("patched total debit", totalDebit);
        assertLe(totalDebit, r.bounds.maxGrossAccountDebitAssets, "patched: debit within signed bound");
    }

    /// GUARDED SIBLING: the same bound on a close commitment from the same aged state is enforced by the Book.
    function test_sibling_CloseCommitWithSameBoundReverts() public {
        Ctx memory c = _agedPosition();
        OrderV3Types.OrderRequest memory r = _v3Request("close", CfdTypes.Side.LONG, BASE_SIZE, 0, true);
        r.bounds.maxGrossAccountDebitAssets = c.bound;
        r.bounds.maxActionChargeAssets = c.bound;
        vm.expectPartialRevert(IOrderLifecycleBook.OrderLifecycleBook__CommitmentBoundExceeded.selector);
        vm.prank(TRADER);
        router.commitOrder(r);
        assertEq(c.custodyBefore, clearinghouse.balanceAssets(TRADER), "close path: nothing debited");
    }

}
```

**Output**:
```
Ran 3 tests for test/perps/poc/IncreaseOpenCarryBounds.t.sol:IncreaseOpenCarryBoundsTest
[PASS] test_patched_DeferredCarryIsBoundedAndNoExcessDebit() (gas: 4061967)
Logs:
  pending carry (USDG atoms) 139921975
  bounty 100000
  fee 400000
  trader bound maxGross/maxActionCharge 600000
  patched terminal reason 8
  patched total debit 100000

[PASS] test_sibling_CloseCommitWithSameBoundReverts() (gas: 3507883)
Logs:
  pending carry (USDG atoms) 139921975
  bounty 100000
  fee 400000
  trader bound maxGross/maxActionCharge 600000

[PASS] test_vulnerable_IncreaseOpenCarryExceedsGrossDebitAndActionChargeBounds() (gas: 4766526)
Logs:
  pending carry (USDG atoms) 139921975
  bounty 100000
  fee 400000
  trader bound maxGross/maxActionCharge 600000
  carry debited at commit 139921975
  carry recorded by Book 0
  total account debit for the order 140422090
  receipt.grossAccountDebitAssets 500115
  receipt.commitment.carryCollectedAssets 0

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 25.23ms (23.99ms CPU time)
Ran 1 test suite in 51.02ms (25.23ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```


