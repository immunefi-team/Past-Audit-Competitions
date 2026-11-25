# \#99974 [SC-Low] Unconditional `engine.creditBounty` call on self-triggered position protection prematurely strips PnL pledge, bypasses `MED-9` `closeLimits`, and corrupts `OrderFinalized` receipts

**Submitted on Nov 15th 2026 at 23:37:04 UTC by @hunraj for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99974
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
When a trader (or an autonomous agent / session key operating on the trader's smart account) triggers their own armed Take-Profit or Stop-Loss protection via `PositionProtectionBook.triggerPositionProtection`, `OrderLiquidationBatchLogic.updateMarkPrice` unconditionally routes the trigger bounty through `engine.creditBounty(plan.account, refundRecipient, ...)` instead of `clearinghouse.releaseReservedExecutionBountyToSource(...)`. Because `refundRecipient == plan.account`, `engine.creditBounty` immediately checkpoints and deducts all accrued carry from the trader's active position margin (`pnlPledgeAssets`) during the trigger transaction, while leaving `pendingIntents[linkedOrderId].commitment.carryCollectedAssets` at `0`. This causes three severe impacts: (1) it completely bypasses the `CloseFinancialLimits` (`maxGrossAccountDebitAssets` and `maxActionChargeAssets`) introduced to fix Cecuro `MED-9`, (2) it prematurely drains active `pnlPledgeAssets` while the position is still sitting open in the FIFO queue (exposing the trader to immediate liquidation before their close can execute), and (3) it corrupts the authenticated `OrderFinalized` receipt economics.


## Vulnerability Details


### 1. The Intended Design (The Smoking Gun in `MarginClearinghouse` and `OrderRouter`)
Whenever a reserved order bounty or protection-trigger bounty is returned to the same account that funded it (`recipient == account`), the protocol is explicitly designed to reclassify the reservation in the clearinghouse **without** calling `engine.creditBounty`, so that an unrecorded carry checkpoint is not triggered on the account's live position.


This is explicitly documented in the NatSpec for `MarginClearinghouse.releaseReservedExecutionBountyToSource` (`packages/perps/src/MarginClearinghouse.sol`, ):
```solidity
/// @notice Releases a settled order or protection-trigger bounty back to the same account that funded it.
/// @dev Callable only by the Engine-reported Router. Unlike the general Engine bounty-credit path, this exact
/// self-transfer changes only the reserved classification and deliberately skips carry checkpointing. Atomic
/// protection cancellation uses this before bounded close carry collection.
function releaseReservedExecutionBountyToSource(
   address account,
   uint256 executionBountyAssets
) external onlyOrderRouter {
   if (executionBountyAssets == 0) {
       return;
   }
   _requireActionReserveDecreaseAboveProtectedFloor(account, executionBountyAssets);
   _unlockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, executionBountyAssets);
}
```


In `OrderRouter._settleTerminalBounty` (`packages/perps/src/OrderRouter.sol`, ), the developers properly implemented this check when settling an order bounty:
```solidity
if (bountyRecipient == order.account) {
   clearinghouse.releaseReservedExecutionBountyToSource(order.account, bountyAssets);
} else {
   engine.creditBounty(
       order.account, bountyRecipient, bountyAssets, accountingPrice, accountingPublishTime
   );
}
```


### 2. The Bug in `OrderLiquidationBatchLogic.updateMarkPrice`
When `PositionProtectionBook.triggerPositionProtection(protectionId, pythUpdateData)` is called (`packages/perps/src/PositionProtectionBook.sol`,), it forwards the caller (`msg.sender`) to the Router as `refundRecipient`:
```solidity
bytes memory refreshCall = abi.encodeWithSelector(UPDATE_MARK_PRICE_SELECTOR, pythUpdateData);
bytes memory callData = bytes.concat(refreshCall, abi.encode(msg.sender, uint256(protectionId)));
linkedOrderId = IPositionProtectionRouterHost(ROUTER).nextCommitId();
_callRouterNoReturn(callData, msg.value);
```


Inside `OrderLiquidationBatchLogic.updateMarkPrice` (`packages/perps/src/router/OrderLiquidationBatchLogic.sol`, lines 499–512), the developers forgot the `if (refundRecipient == plan.account)` check and called `engine.creditBounty` unconditionally:
```solidity
if (isProtectionTrigger) {
   uint64 linkedOrderId = host.nextCommitId();
   IPositionProtectionBook.TriggerPlan memory plan =
       protectionBook.activate(protectionId, snapshot.markPrice, snapshot.publishTime, linkedOrderId);
   _registerProtectionTrigger(host, protectionId, linkedOrderId, plan);
   _queueProtectionAttempt(host, linkedOrderId, plan.account, plan.side, plan.size);
   engine.creditBounty(
       plan.account,
       refundRecipient, // <--- When self-triggered, refundRecipient == plan.account!
       plan.triggerBountyAssets,
       snapshot.markPrice,
       snapshot.publishTime
   );
}
```


### 3. Step-by-Step Execution Trace of the Bug
When `plan.account` (either the trader directly or an autonomous agent / session key operating on the trading smart account, as documented in `WORKING_WITH_AI_AGENTS.md`) calls `triggerPositionProtection`:
1. `refundRecipient` is `plan.account`.
2. `_registerProtectionTrigger` and `_queueProtectionAttempt` register the generated close order (`linkedOrderId`) in `OrderLifecycleBook` and queue it in `OrderRouter`. Because protection triggers do not call `OrderLifecycleBook.recordCommitment`, `pendingIntents[linkedOrderId].commitment.carryCollectedAssets` is **`0`**.
3. `engine.creditBounty(plan.account, plan.account, plan.triggerBountyAssets, ...)` is called (`packages/perps/src/CfdEngine.sol`, lines 473–496).
4. Because `_positions[plan.account].lots > 0`, `creditBounty` calls `_checkpointBountyRecipient(plan.account, ...)` -> `_realizeCarryFromSettlement(plan.account, pos)`.
5. `_realizeCarryFromSettlement` immediately collects **all accrued carry** from `plan.account`'s `pnlPledgeAssets` (active position margin) and `freeSettlementAssets`, transfers that USDG to `HousePool`, and resets `pos.lastCarryIndex` to the current side carry index.
6. When `linkedOrderId` is subsequently evaluated and executed in a later block, `CfdOrderPolicyEvaluator.assessCommittedOrder` computes the order's carry and gross debit starting from the already-reset `pos.lastCarryIndex` and `pending.commitment.carryCollectedAssets == 0`.


## Impact Details


1. **Complete Bypass of `CloseFinancialLimits` (Broken Fix for Cecuro `MED-9`):**
  To fix Cecuro `MED-9` (*"Keepers can retry protection closes without the trader's financial limits"*), the team added `CloseFinancialLimits` (`maxExecutionNotionalAssets`, `maxGrossAccountDebitAssets`, `maxActionChargeAssets`, `maxExplicitFeesAssets`) to `PositionProtectionBook`. As documented in `WORKING_WITH_AI_AGENTS.md`: *"These limits apply per attempt, including commitment charges."*
  Because `engine.creditBounty` strips all accrued carry during `triggerPositionProtection` without recording it in `pending.commitment.carryCollectedAssets`, `CfdOrderPolicyEvaluator.assessCommittedOrder` is completely blind to the carry debit. The close order executes and succeeds even when the actual carry + close charges vastly exceed the account's `maxGrossAccountDebitAssets` and `maxActionChargeAssets` ceilings.


2. **Premature Margin Drain Exposes the Position to Immediate Liquidation Before Close Execution:**
  A triggered protection close (`linkedOrderId`) cannot execute in the same block it is triggered (`ExecutionBlockNumberLib.current() == order.commitBlock` returns `PendingReason.SameBlock`); it must sit in the FIFO queue until a later block and post-trigger Pyth tick.
  When a third-party keeper triggers the protection (`refundRecipient != plan.account`), `plan.account`'s `pnlPledgeAssets` is not touched at trigger time. However, when the trader or their agent self-triggers, `_realizeCarryFromSettlement` physically rips the accrued carry out of `pnlPledgeAssets` during `triggerPositionProtection` while the position is still open. Even worse, because `_checkpointBountyRecipient` runs *before* `transferReservedSettlement` unlocks the `$0.20` trigger bounty, that bounty cannot even cushion the debit. If the position is near maintenance margin (such as during a stop-loss trigger or a Friday FAD margin jump), this premature `pnlPledgeAssets` reduction can make the account immediately liquidatable in the window before `linkedOrderId` can execute.


3. **Corrupted `OrderFinalized` Lifecycle Receipt Evidence:**
  In `OrderRouterExecutionSidecar._finalizeExecutedOrder`, the terminal `OrderFinalized` receipt is populated from `assessment` and the pre/post-execution account balances. Because the carry was deducted during `triggerPositionProtection` rather than during order commitment or execution, the authenticated `OrderFinalized` receipt permanently under-reports `economics.carryAssets`, `actionChargeAssessedAssets`, `actionChargeCollectedAssets`, and `grossAccountDebitAssets`, and records the already-debited balance as `preSettlementBalanceAssets`.


***


> 1. *"Is self-triggering a realistic flow?"* — Yes. Both `WORKING_WITH_AI_AGENTS.md` and `DEPLOYMENT.md` explicitly instruct traders and autonomous smart-account agents that they can trigger and execute orders themselves rather than relying solely on third-party keepers. In smart-account architectures (where session keys have bounded permissions and rely on on-chain `closeLimits` enforced by the contract to prevent excessive debits), self-triggering by the agent is a primary operating mode.
> 2. *"Was this intentional?"* — No. As shown in `MarginClearinghouse.sol`, the developers explicitly wrote `releaseReservedExecutionBountyToSource` for *"a settled order or protection-trigger bounty"* to *"deliberately skip carry checkpointing"* on self-transfers, and used it in `OrderRouter.sol:` (`_settleTerminalBounty`) and `OrderLiquidationBatchLogic.sol:` (`commitProtectedClose`), but simply forgot the `if (refundRecipient == plan.account)` branch in `OrderLiquidationBatchLogic.sol:505` (`updateMarkPrice`).
***


## Recommendation
In `OrderLiquidationBatchLogic.updateMarkPrice` (`packages/perps/src/router/OrderLiquidationBatchLogic.sol`,), mirror the self-bounty handling used in `OrderRouter._settleTerminalBounty` by calling `releaseReservedExecutionBountyToSource` when `refundRecipient == plan.account`:


```diff
        if (isProtectionTrigger) {
            uint64 linkedOrderId = host.nextCommitId();
            IPositionProtectionBook.TriggerPlan memory plan =
                protectionBook.activate(protectionId, snapshot.markPrice, snapshot.publishTime, linkedOrderId);
            _registerProtectionTrigger(host, protectionId, linkedOrderId, plan);
            _queueProtectionAttempt(host, linkedOrderId, plan.account, plan.side, plan.size);
-            engine.creditBounty(
-                plan.account,
-                refundRecipient,
-                plan.triggerBountyAssets,
-                snapshot.markPrice,
-                snapshot.publishTime
-            );
+            if (refundRecipient == plan.account) {
+                IMarginClearinghouse(engine.clearinghouse())
+                    .releaseReservedExecutionBountyToSource(plan.account, plan.triggerBountyAssets);
+            } else {
+                engine.creditBounty(
+                    plan.account,
+                    refundRecipient,
+                    plan.triggerBountyAssets,
+                    snapshot.markPrice,
+                    snapshot.publishTime
+                );
+            }
        }
```


## References
* `packages/perps/src/router/OrderLiquidationBatchLogic.sol`:  (`updateMarkPrice` unconditional `engine.creditBounty` call).
* `packages/perps/src/OrderRouter.sol`:  (`_settleTerminalBounty` proper `bountyRecipient == order.account` check).
* `packages/perps/src/MarginClearinghouse.sol`: (`releaseReservedExecutionBountyToSource` NatSpec and implementation).
* `packages/perps/src/CfdEngine.sol`:  (`creditBounty` and `_checkpointBountyRecipient`).


## Proof of Concept

## Proof of Concept
Place the following test in `packages/perps/test/perps/spec/lp/SelfTriggerBypassExploit.t.sol` and run:
```bash
forge test --match-test test_PoC_SelfTriggerBypassesCloseLimitsAndDeductsCarry -vvv
```


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;


import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";


contract SelfTriggerBypassExploitTest is BasePerpTest {
   address trader = address(0x1111);
   address keeper = address(0x2222);


   function test_PoC_SelfTriggerBypassesCloseLimitsAndDeductsCarry() public {
       IPositionProtectionBook protectionBook = router.positionProtectionBook();


       // 1. Setup: Fund trader and open a LONG position at $1.00
       _fundTrader(trader, 5000e6);
       _open(trader, CfdTypes.Side.LONG, 100_000e18, 2000e6, 1e8);


       // Initialize PletherOracle's accepted live basket so createPositionProtection can read getLatestPrice
       router.updateMarkPrice(_mockPythUpdateData(1e8));


       // 2. Trader arms LONG TP protection at $0.90 (LONG profits when mark falls)
       // Close execution fee at $0.90 is 36(36e6).30daysofcarryis~39.48 (39.48e6).
       // True total action charge & gross debit = ~$75.48 (75.48e6).
       // We set a strict $50 (50e6) limit on both action charges and gross debit.
       PositionProtectionTypes.PositionProtectionParams memory params;
       params.takeProfitTriggerPrice = 0.9e8;
       params.stopLossTriggerPrice = 0;
       params.closeLimits.maxExecutionNotionalAssets = type(uint256).max;
       params.closeLimits.maxGrossAccountDebitAssets = 50e6;
       params.closeLimits.maxActionChargeAssets = 50e6;
       params.closeLimits.maxExplicitFeesAssets = type(uint256).max;


       vm.prank(trader);
       uint64 protectionId = protectionBook.createPositionProtection(params);


       // 3. Fast-forward 30 days and 1 block to accrue ~$39.48 in carry
       vm.warp(vm.getBlockTimestamp() + 30 days);
       vm.roll(vm.getBlockNumber() + 1);


       // 4. SELF-TRIGGER: Trader (or their autonomous smart-account agent) triggers the TP at $0.90
       bytes[] memory triggerUpdate = _mockPythUpdateData(0.9e8);


       uint256 pledgeBeforeTrigger = clearinghouse.pnlPledgeAssets(trader);
       uint256 expectedCarry = _expectedIndexedCarryAssets(trader);
       assertGt(expectedCarry, 35e6, "Accrued carry (~$39.48) + $36 fee exceeds the $50 limit");


       vm.prank(trader);
       uint64 linkedOrderId = protectionBook.triggerPositionProtection(protectionId, triggerUpdate);


       // --- VERIFY IMPACT 1: PREMATURE UNRECORDED CARRY DEDUCTION ---
       uint256 pledgeAfterTrigger = clearinghouse.pnlPledgeAssets(trader);
       assertEq(
           pledgeBeforeTrigger - pledgeAfterTrigger,
           expectedCarry,
           "Carry was immediately ripped from PnL pledge during triggerPositionProtection"
       );
       assertEq(
           router.lifecycleBook().pendingIntent(linkedOrderId).commitment.carryCollectedAssets,
           0,
           "Lifecycle book recorded 0 commitment carry despite the deduction"
       );


       // --- VERIFY IMPACT 2: EVALUATOR SEES 0 TRIGGER CARRY BEFORE EXECUTION ---
       OrderV3Types.ExecutionAssessment memory assessment =
           policyEvaluator.assessCommittedOrder(address(engine), linkedOrderId, keeper, 0.9e8);
       assertEq(
           assessment.carryAssets,
           0,
           "Evaluator is completely blind to the carry deducted during trigger"
       );


       // 5. KEEPER EXECUTES THE QUEUED CLOSE IN THE NEXT BLOCK
       vm.roll(vm.getBlockNumber() + 1);
       bytes[] memory execUpdate = _mockPythUpdateData(0.9e8);


       vm.prank(keeper);
       OrderV3Types.ExecutionResult memory result = router.executeOrder(linkedOrderId, execUpdate);


       // --- VERIFY IMPACT 3: COMPLETE BYPASS OF CLOSE LIMITS ---
       // Execution SUCCEEDS even though total carry (~39.48)+fee(36) = ~$75.48 > $50 limit!
       assertEq(
           uint8(result.status),
           uint8(OrderV3Types.LifecycleStatus.Executed),
           "Close executed successfully, bypassing maxActionChargeAssets and maxGrossAccountDebitAssets!"
       );
   }
}
```

