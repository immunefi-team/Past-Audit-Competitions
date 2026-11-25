# \#99665 [SC-Low] Open/increase commitment collects carry but reports it as zero, silently bypassing the trader's pinned gross-debit and action-charge caps

**Submitted on Nov 14th 2026 at 18:13:02 UTC by @maigadoh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99665
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

```markdown
## Summary
The external V3 request sets "inclusive maxima for execution bounty, execution notional, gross account debit, assessed action charge" that a trader relies on to cap what one commitment can charge them ([SECURITY.md#L755-L757](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L755-L757)), and `OrderLifecycleBook.recordCommitment` enforces both `maxGrossAccountDebitAssets` and `maxActionChargeAssets` against `effects.carryCollectedAssets` for any order type. For an open/increase, `_reserveExecutionBounty` hand-builds `effects` with `carryCollectedAssets = 0` while the bounty-lock and margin-reserve calls that follow realize the account's real carry, so the Book checks zero and both pinned limits are bypassed. In the PoC an increase pinning `maxActionChargeAssets = 0` and `maxGrossAccountDebitAssets = 0.20 USDG` is debited 37.12 USDG of carry at commit, while the identical-state close is rejected with `OrderLifecycleBook__CommitmentBoundExceeded`.

## Root Cause
`_reserveExecutionBounty` returns Engine-measured effects for a close, but for an open/increase it builds the effects by hand with only the bounty and leaves `carryCollectedAssets = 0`. The bounty-lock here, and the margin reserve in the caller, each realize the account's accrued carry through `_checkpointCarryBeforeMarginChange`, but the hand-built effects are never updated before the Book checks them.

[OrderReservationAccounting.sol#L114-L128](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L114-L128)

```solidity
if (isClose) {
    ...
    return _reserveCloseExecutionBounty(account, sizeDelta); // measured: carryCollectedAssets filled
}
effects.bountyFromFreeAssets = bounty;
// vulnerable: carryCollectedAssets stays 0 for opens/increases
...
clearinghouse.lockReservedSettlement(account, bounty); // realizes carry via _checkpointCarryBeforeMarginChange
```

[OrderCommitHandler.sol#L27-L33](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L27-L33)

```solidity
CfdEnginePlanTypes.CloseCommitment memory effects =
    _reserveExecutionBounty(account, order.sizeDelta, executionBountyAssets, order.isClose);
...
_onOrderCommitted(order, effects); // Book checks zero carry
_reserveCommittedMargin(account, orderId, order.isClose, order.marginDelta); // also realizes carry, after the check
```

[OrderLifecycleBook.sol#L335-L347](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L335-L347)

```solidity
uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;
if (gross > pending.bounds.maxGrossAccountDebitAssets) { revert ... }           // reads 0 + bounty for an open
if (effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets) { revert ... } // reads 0 for an open
```

The check is order-type-agnostic and the close path proves the intent: `reserveCloseOrderExecutionBounty` fills `carryCollectedAssets` with the measured carry, so a close with `maxActionChargeAssets = 0` reverts. The spec states the same requirement, "Collected commitment carry is included exactly once in gross account debit, assessed/collected action charges, and carry totals for the user's financial bounds" ([ACCOUNTING_SPEC.md#L943-L944](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L943-L944)). The execution-time evaluator does not recover the gap either: `_includeCommitment`, which folds the committed carry back into the gross-debit and action-charge checks, runs only for closes ([CfdOrderPolicyEvaluator.sol#L65-L67](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluator.sol#L65-L67)).

## Attack Path
Preconditions: the trader already holds an open position (a brand-new open accrues no carry, since `realizeCarryBeforeMarginChange` no-ops when `lots == 0`), and the order is an increase funding a nonzero bounty or margin, which is every ordinary increase. No attacker and no privileged role.

1. A trader holds LONG 100,000 tokens with 5,000 USDG margin, opened and executed under the release risk parameters.
2. 30 days pass, so carry accrues on the position (37.12 USDG at the release `baseCarryBps = 500`).
3. A permissionless keeper refreshes the oracle (`router.updateMarkPrice`) so the market is not close-only, as in normal operation.
4. The trader commits an increase of 10,000 tokens pinning `maxActionChargeAssets = 0` and `maxGrossAccountDebitAssets = 0.20 USDG` (the open-order bounty, which caps at `maxOpenOrderExecutionBountyAssets`).
5. The bounty lock and margin reserve realize 37,124,075 atoms of carry; `recordCommitment` receives `carryCollectedAssets = 0` and passes. Custody falls by 37.12 USDG and `pendingIntent(orderId).commitment.carryCollectedAssets == 0`.
6. Control: the same trader, in the same state, commits a partial close pinning `maxActionChargeAssets = 0`. The Engine fills the real carry into the effects and `recordCommitment` reverts `OrderLifecycleBook__CommitmentBoundExceeded(ActionCharge, 37124075, 0)`.

The one alternative protection, `minPostSettlementBalanceAssets`, cannot be used here: `_validateFreshRequest` rejects it for any order on an existing position with `OrderRouter__SettlementFloorUnsupported` ([OrderLiquidationBatchLogic.sol#L307-L312](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L307-L312)), so for an increase the two bypassed bounds are the only per-order charge protection.

## Impact
The documented per-order gross-debit and action-charge ceilings do not hold for opens/increases: the PoC debits 37.32 USDG of gross charge (the 0.20 USDG bounty plus 37.12 USDG of carry) against a 0.20 USDG limit (a 187x overrun) and 37.12 USDG of action charge against a zero limit, and the overrun grows without bound as carry accrues, with no alternative protection available for an increase. The committed carry is owed and moves to the LP pool, so no value is taken by a wrong party; this is a broken guarantee ("Contract fails to deliver promised returns, but doesn't lose value"), not theft. It is not a design choice (the generic `recordCommitment` check and the close control both enforce the bound) and not a known issue (Cecuro MED-22 is carry rounding, a different root cause).

**Likelihood:** High — PoC-proven under the release config; any trader with a position, an ordinary increase, and a permissionless keeper refresh; no privileged actor or special state.
**Severity:** Low — the pinned inclusive maxima for gross account debit and assessed action charge are silently ignored on opens/increases; the trader is over-charged owed carry past a limit they set, but no value is lost to a third party.

## Mitigation
In `_createPendingOrder`, read `clearinghouse.balanceAssets(account)` before reservation and again after `_reserveCommittedMargin`, set `effects.carryCollectedAssets` and the before/after fields from the difference (the measured drop equals the realized carry, whether taken from free settlement or position pledge), then call `_onOrderCommitted`; if Router initcode has no room, move the measurement into `recordCommitment`.
```

## Proof of Concept

```markdown
## Proof of Concept
Add the PoC below as a new test file `packages/perps/test/perps/spec/trader/OpenCommitCarryBypassPoc.t.sol`. It extends the protocol's own `BasePerpTest` harness, uses the release risk parameters from `packages/perps/DEPLOYMENT.md` (default config), drives the real `OrderRouter` / `MarginClearinghouse` / `CfdEngine` stack, and mocks only the Pyth feed. Then run:

```bash
forge test --match-path 'test/perps/spec/trader/OpenCommitCarryBypassPoc.t.sol' --match-test test_OpenIncreaseCommitCarryBypassesPinnedBounds -vvvv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {console2} from "forge-std/console2.sol";

contract OpenCommitCarryBypassPoc is BasePerpTest {

    address private constant TRADER = address(0x7AAD);
    address private constant KEEPER = address(0xBEEF);
    uint256 private constant PRICE = 100_000_000; // 1.0, 8 decimals

    // packages/perps/DEPLOYMENT.md release risk parameters (default config).
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = CfdTypes.RiskParams({
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

    function _request(
        CfdTypes.Side side,
        bool close,
        uint256 size,
        uint256 margin
    ) private view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = keccak256(abi.encode("carry-bypass-poc", router.nextCommitId()));
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.isClose = close;
        r.targetPrice = (side == CfdTypes.Side.LONG) == close ? 2e8 : 1;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
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

    function test_OpenIncreaseCommitCarryBypassesPinnedBounds() public {
        // Precondition: a live LONG position opened and executed under the release config.
        _fundTrader(TRADER, 11_000e6);
        OrderV3Types.OrderRequest memory openReq = _request(CfdTypes.Side.LONG, false, 100_000e18, 5_000e6);
        vm.prank(TRADER);
        uint64 openId = router.commitOrder(openReq);
        bytes[] memory openData = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory openResult = router.executeOrder(openId, openData);
        assertEq(uint8(openResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executed");
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 100_000e18, "base LONG position established");

        // 30 days pass; carry accrues on the open position.
        vm.warp(vm.getBlockTimestamp() + 30 days);

        // A permissionless keeper refreshes the oracle so the market is not close-only.
        vm.prank(KEEPER);
        router.updateMarkPrice(_mockPythUpdateData(PRICE));

        uint256 balanceBefore = clearinghouse.balanceAssets(TRADER);

        // Probe: measure the open bounty and the carry an increase will collect, then roll back.
        uint256 probe = vm.snapshotState();
        OrderV3Types.OrderRequest memory probeReq = _request(CfdTypes.Side.LONG, false, 10_000e18, 1_000e6);
        vm.prank(TRADER);
        uint64 pid = router.commitOrder(probeReq);
        uint256 openBounty =
            clearinghouse.getBountyReservation(IMarginClearinghouse.BountyKind.Order, pid).amountAssets;
        uint256 probeCarry = balanceBefore - clearinghouse.balanceAssets(TRADER);
        uint256 probeRecorded = router.lifecycleBook().pendingIntent(pid).commitment.carryCollectedAssets;
        vm.revertToStateAndDelete(probe);

        // Control: an identical-state partial CLOSE pinning maxActionChargeAssets = 0 is rejected.
        uint256 ctl = vm.snapshotState();
        OrderV3Types.OrderRequest memory c = _request(CfdTypes.Side.LONG, true, 50_000e18, 0);
        c.bounds.maxActionChargeAssets = 0;
        vm.prank(TRADER);
        vm.expectPartialRevert(IOrderLifecycleBook.OrderLifecycleBook__CommitmentBoundExceeded.selector);
        router.commitOrder(c);
        vm.revertToStateAndDelete(ctl);

        // Exploit: the increase pins the SAME zero action charge and a gross-debit cap equal to the bounty,
        // yet commits and silently debits the carry.
        OrderV3Types.OrderRequest memory inc = _request(CfdTypes.Side.LONG, false, 10_000e18, 1_000e6);
        inc.bounds.maxActionChargeAssets = 0;
        inc.bounds.maxGrossAccountDebitAssets = openBounty;
        vm.prank(TRADER);
        uint64 incId = router.commitOrder(inc);

        uint256 realCarry = balanceBefore - clearinghouse.balanceAssets(TRADER);
        uint256 recorded = router.lifecycleBook().pendingIntent(incId).commitment.carryCollectedAssets;

        console2.log("pinned maxActionChargeAssets (atoms)   :", uint256(0));
        console2.log("pinned maxGrossAccountDebitAssets      :", inc.bounds.maxGrossAccountDebitAssets);
        console2.log("open-order bounty (atoms)              :", openBounty);
        console2.log("carry actually debited from custody    :", realCarry);
        console2.log("carry recorded in lifecycle Book       :", recorded);
        console2.log("true gross debit (bounty + carry)      :", openBounty + realCarry);

        assertEq(probeCarry, realCarry, "probe and exploit collect the same carry");
        assertEq(probeRecorded, 0, "probe increase also recorded zero carry");
        assertGt(realCarry, 0, "real carry left the trader's custody at commit");
        assertEq(recorded, 0, "lifecycle Book recorded zero carry for the increase");
        assertGt(
            openBounty + realCarry,
            inc.bounds.maxGrossAccountDebitAssets,
            "true gross debit exceeds the pinned maximum that recordCommitment accepted"
        );
    }

}
```

Output:

```bash
[PASS] test_OpenIncreaseCommitCarryBypassesPinnedBounds() (gas: 6390510)
Logs:
  pinned maxActionChargeAssets (atoms)   : 0
  pinned maxGrossAccountDebitAssets      : 200000
  open-order bounty (atoms)              : 200000
  carry actually debited from custody    : 37124075
  carry recorded in lifecycle Book       : 0
  true gross debit (bounty + carry)      : 37324075
```

The `-vvvv` trace shows the asymmetry on the same order id in the same state. Increase: `CfdEngine` emits `CarryRealized(... realizedCarryAssets: 37124075, marginConsumedAssets: 37124075 ...)`, then `OrderLifecycleBook::recordCommitment(2, CloseCommitment({ carryCollectedAssets: 0, ... settlementBeforeAssets: 0, settlementAfterAssets: 0 }))` and the commit succeeds. Close: `recordCommitment(2, CloseCommitment({ carryCollectedAssets: 37124075, settlementBeforeAssets: 10909899801, settlementAfterAssets: 10872775726, ... }))` then `Revert OrderLifecycleBook__CommitmentBoundExceeded(4, 37124075, 0)` (constraint 4 = ActionCharge).
```
