# \#97971 [SC-Low] Self-triggered position protection collects accrued carry outside the trader's retained close limits and the close's authenticated receipt

**Submitted on Nov 4th 2026 at 05:40:51 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97971
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Position protection (take-profit/stop-loss) stores four trader-chosen financial ceilings and copies them into every generated close attempt. This is the remediation of audit finding MED-9 ("Keepers can retry protection closes without the trader's financial limits").

Triggering is permissionless, so the protected trader, or their trading smart account, can trigger their own protection. When they do, the Router pays the 0.20 USDG trigger bounty from the trader back to the trader through `CfdEngine.creditBounty(account, account, ...)`. That function checkpoints the beneficiary's carry, so it collects **all of the trader's accrued carry inside the trigger transaction**. This happens after the generated close has been queued with a zero commitment, and outside every check that enforces the retained ceilings. The close then executes as if no carry had been charged, and its authenticated receipt records neither the debit nor a commitment.

In the PoC (release risk parameters, public entrypoints only), the protection allows at most 10 USDG of gross account debit and action charge:

- The self-trigger debits **301.753575 USDG** of carry and does not revert.
- The generated close executes. Its verified receipt reports a gross account debit and carry of **0.000124 USDG** and a commitment carry of 0.
- The same protection with the same carry, **triggered by a third party**, is held to the ceiling. The close fails with `GrossAccountDebit` (actual 301.753700 vs limit 10 USDG) and the protection latches.
- The owner's atomic exit, `cancelPositionProtectionAndCommitClose`, with the same ceiling reverts with `CommitmentBoundExceeded(GrossAccountDebit, 301753575, 10000000)`.

The carry is owed and goes to LPs, so no value is lost. What fails is the documented guarantee that the retained ceilings bound the protection close "including commitment charges", and that the close's receipt records its economics. Severity: **Low**.

## Vulnerability Details

### Why this is not intended behaviour

1. **The protocol has an explicit carry-free rule for paying a bounty back to its own source, and every other bounty payout follows it.**
   - When an order's bounty recipient is the order's own account, `OrderRouter._settleTerminalBounty` does not call `creditBounty`. It calls `MarginClearinghouse.releaseReservedExecutionBountyToSource` (`OrderRouter.sol` L336-L341).
   - That helper's NatSpec names this exact case: "Releases a settled order or **protection-trigger bounty** back to the same account that funded it. ... Unlike the general Engine bounty-credit path, this exact self-transfer changes only the reserved classification and **deliberately skips carry checkpointing**" (`MarginClearinghouse.sol` L716-L720).
   - The interface gives the reason: "because settlement custody does not change and the bounty returns to its source" (`IMarginClearinghouse.sol` L371-L374).
   - The integration guide says: "Successful self-execution releases it to the account's free settlement; external execution credits the keeper's Clearinghouse account" (`WORKING_WITH_AI_AGENTS.md` L231-L232).
   - The protection trigger is the only bounty payout without this self-recipient branch.
2. **The owner's other protection exit applies that rule so that carry stays inside the close limits.**
   - `commitProtectedClose`, behind `cancelPositionProtectionAndCommitClose`, releases the same trigger-bounty classification without a checkpoint. Its code comment: "Release it without a carry checkpoint so all commitment carry is collected, bounded and recorded by the ordinary close path below" (`OrderLiquidationBatchLogic.sol` L176-L179).
   - `SECURITY.md` L535-L543: "The Router releases the unused trigger bounty without checkpointing carry before ordinary close commitment. All commitment carry is then collected and recorded by the canonical close path, including its gross-debit and action-charge bounds ... but does not bypass any execution limit."
3. **The protection documentation says the ceilings cover commitment charges, and that triggering reverts if they would be exceeded.**
   - `WORKING_WITH_AI_AGENTS.md` L282-L283: "These limits apply per attempt, including commitment charges, not cumulatively across retries. If commitment would exceed a limit, triggering reverts and the owner can replace the `Armed` protection."
   - In the PoC, the trigger transaction charges 301.75 USDG against a 10 USDG limit, and triggering succeeds.
4. **Protection carry is specified to be assessed when the close executes, not collected at trigger.**
   - `ACCOUNTING_SPEC.md` L954: "Protection trigger/retry | Queue through the protection path without ordinary commitment carry effects | Append a fresh authenticated zero record; carry is assessed at execution."
   - `CANONICAL_ENTRYPOINTS.md` L112-L113: "A trigger or retry queues a new close without ordinary commitment carry effects; its execution still settles carry."
   - `CANONICAL_ENTRYPOINTS.md` L118-L126: the four limits are ceilings "for future triggered or retried close attempts".
   - All of this holds for a third-party trigger. It does not hold for a self-trigger.
5. **This is the residual of MED-9.**
   - MED-9's stated impact: "the protected trader can be closed after carry, fees, or frozen spread exceed the limits they could set on an ordinary close".
   - The team's resolution: "The protection retains trader-selected ceilings ... and copies them into both the initial close and every permissionless retry."
   - The retained ceilings arrived with that fix. The `creditBounty` call in the trigger path is older. The fix did not account for the trigger itself collecting carry when the caller is the protected account.

No document, NatSpec comment or known issue says that self-triggering exempts accrued carry from the retained ceilings or from the close receipt. No project test exercises a trader triggering their own protection: every trigger in the protection specs, the keeper batch spec and the protection bounty state machine uses a separate keeper address. The state machine also sets `baseCarryBps = 0`.

### Root cause

**1. The trigger names its caller as the bounty beneficiary.** `PositionProtectionBook.triggerPositionProtection` (L370-L378) appends `msg.sender` to the delegated Router call as the refund/bounty recipient.

**2. The Router queues a zero-commitment child, then pays the bounty through the checkpointing path.** From `OrderLiquidationBatchLogic.updateMarkPrice` (L583-L622):

```solidity
if (isProtectionTrigger) {
    uint64 linkedOrderId = host.nextCommitId();
    IPositionProtectionBook.TriggerPlan memory plan =
        protectionBook.activate(protectionId, snapshot.markPrice, snapshot.publishTime, linkedOrderId);
    _registerProtectionTrigger(host, protectionId, linkedOrderId, plan);              // ceilings copied into the child
    _queueProtectionAttempt(host, linkedOrderId, plan.account, plan.side, plan.size); // zero bounty, no commitment
    engine.creditBounty(                                   // self-trigger: refundRecipient == plan.account
        plan.account, refundRecipient, plan.triggerBountyAssets, snapshot.markPrice, snapshot.publishTime
    );
}
```

- `_registerProtectionTrigger` registers the child with a zero bounty.
- `_queueProtectionAttempt` reaches `OrderRouter.executePositionProtectionTriggerItem` (L436-L471), which records a zero bounty reservation and the order.
- Neither calls `recordCommitment`, so the child's stored commitment stays all-zero. That is the documented protection behaviour, and it is correct while the trader's carry remains pending.

**3. `creditBounty` collects the beneficiary's carry before moving the bounty.**
- `CfdEngine.creditBounty` (L460-L492) calls `_checkpointBountyRecipient(beneficiary, ...)` (L203-L217).
- For a beneficiary with a position, that runs `_realizeCarryFromSettlement` (L1407-L1449). It takes carry from position margin and then free settlement, transfers it to the HousePool, and books it as LP revenue.
- Only then does `transferReservedSettlement(account, account, bounty)` run. With source and beneficiary equal, that is a reclassification inside one account, so the trader's net loss is exactly the carry.

**4. The ceilings and the receipt only see the execution-time assessment.**
- The retained ceilings are enforced by the policy evaluator at execution, against a snapshot taken after the trigger-time debit.
- Commitment effects are added for closes (`CfdOrderPolicyEvaluator.sol` L64-L67, then `_includeCommitment` in `CfdOrderPolicyEvaluatorBase.sol` L474-L487), but for a protection child they are zero.
- The receipt copies that zero commitment (`OrderRouterExecutionSidecar.sol` L717) and the execution-only assessment (L781-L799). Its `preSettlementBalanceAssets` is sampled after the debit.

With a third-party trigger, step 3 checkpoints the keeper's own account, which is documented and correct. The protected trader's carry stays pending until the child executes, where the retained ceiling applies to it. The PoC uses this as its control.

### Sequence (PoC numbers)

1. A trader holds a 300,000 USDG LONG with 20,000 USDG margin (about 15x leverage against a 500x maximum).
2. They arm a take-profit at 0.90 with `maxGrossAccountDebitAssets = maxActionChargeAssets = 10 USDG`. A take-profit's price gain covers its own trading charges, so a near-zero account-debit allowance is a natural choice.
3. Over 28 days, 301.753575 USDG of carry accrues: release `baseCarryBps = 500`, a 1,000,000 USDG Junior pool, about 10.8 USDG per day.
4. The price crosses 0.90 and the trader triggers the protection themselves. The trigger succeeds and collects 301.753575 USDG:
   - the trader's settlement falls from 99,430.698080 to 99,128.944505 USDG;
   - pool assets rise by exactly that amount;
   - the queued child stores the 10 USDG ceilings and a zero commitment.
5. A keeper executes the child. Only 124 atoms of carry have accrued since the trigger, so the close passes its 10 USDG ceilings and executes. The verified `OrderFinalized` receipt reports `grossAccountDebitAssets = 124`, `carryAssets = 124` and `commitment.carryCollectedAssets = 0`.

Controls on the same state:

- **Third-party trigger.** No trader carry is collected at trigger, and the trader pays only the 0.20 USDG bounty. At execution the child fails with `ConstraintViolation` / `GrossAccountDebit` (actual 301.753700, limit 10 USDG). The carry collection rolls back, the protection becomes `Latched` for the owner to amend, and the position stays open.
- **Owner atomic exit.** `cancelPositionProtectionAndCommitClose` with a 10 USDG ceiling reverts with `OrderLifecycleBook__CommitmentBoundExceeded(GrossAccountDebit, 301753575, 10000000)`. Nothing changes and the protection stays `Armed`.
- **Three days of carry (32.330851 USDG).** The self-triggered close still executes. The third-party-triggered one fails at 32.330976 against the 10 USDG limit.

### Self-triggering is an expected path

- `triggerPositionProtection` is permissionless and is listed among the user-facing protection actions (`WORKING_WITH_AI_AGENTS.md` L257-L267).
- The official account-abstraction client ships `buildTriggerProtection`, which returns a `SmartAccountCall` for the trader's smart account (`packages/perps-aa-client/src/keeperExecution.ts` L52-L55).
- `DEPLOYMENT.md` notes that users can execute or expire queued orders through the same public entrypoints as keepers (L402-L403). It also documents a same-account fallback for when keepers, the sponsor or the API are unavailable (L423-L447).
- A trader who runs their own automation, or who acts during a keeper delay, will naturally trigger their own protection, and gets their own 0.20 USDG bounty back.
- Nothing warns that doing so removes accrued carry from the ceilings they set.

### Anticipated objections

- **"Bounty credits collect the beneficiary's carry by design"** (`ACCOUNTING_SPEC.md` L702-L705, `SECURITY.md` L954-L957).
  - That rule covers a keeper credited with someone else's bounty, whose reachable collateral changes.
  - For a payment back to the source, custody does not change, and the protocol's own rule (point 1) is to skip the checkpoint.
  - The Engine unit test `CarryCreditCheckpoint.t.sol::test_SelfBountyProtectsReserveThenReleasesFullCredit` calls `creditBounty(A, A, ...)` directly while pranking the Router, to test reserve protection. In production the Router never routes an order's self-payment through `creditBounty`. The test predates the retained protection ceilings and does not involve them.
- **"The trigger bounty is separate from the close"** (`CANONICAL_ENTRYPOINTS.md` L125-L126).
  - Agreed. The bounty nets to zero here and this report is not about it.
  - The debit is the trader's carry, which the specification assigns to the close's execution (point 4). It is collected early only because the self-payment takes the checkpointing path.
- **"There is no loss; the carry is owed."** Agreed, and nothing more is claimed. A ceiling the trader set, which the protocol documents as binding, is not applied, and the protocol's authenticated receipt misstates the close.

## Impact Details

- **The retained ceilings do not bound a self-triggered protection close.**
  - Whenever a protected trader triggers their own protection, all carry accrued since their last checkpoint is collected outside `maxGrossAccountDebitAssets` and `maxActionChargeAssets`.
  - The amount has no cap and grows linearly with time: about 10.8 USDG per day in the PoC, so 30 times the ceiling after four weeks and already 3.2 times after three days.
  - The outcome of one protection now depends on who sends the trigger transaction:
    - a third-party trigger is held to the ceiling: the close fails and the protection latches for the owner to amend;
    - a self-trigger is not: the close executes after a 301.75 USDG debit.
  - This breaks the guarantee that the MED-9 remediation introduced and documents: "If commitment would exceed a limit, triggering reverts".
- **The authenticated receipt understates the protection close.**
  - For closes, the receipt's commitment fields and gross debit are defined to include commitment-time carry (`ACCOUNTING_SPEC.md` L939-L945; `OrderV3Types.sol` L201-L203: carry "including any commitment-time carry collection added to the assessment").
  - The self-triggered child's receipt reports zero commitment carry and a 0.000124 USDG gross debit. Its `preSettlementBalanceAssets` is already net of the 301.75 USDG collection.
  - The `CarryRealized` event in the trigger transaction still shows the debit, so reconciliation against Engine events (`WORKING_WITH_AI_AGENTS.md` L312) can recover it. The receipt alone cannot, and limit monitoring built on receipts under-reports the cost of the protection close.
- **What is not claimed.**
  - The carry is owed and reaches LPs in full; pool assets rise by exactly the collected amount.
  - Health projections already deduct pending carry (`ACCOUNTING_SPEC.md` L698-L699), so the early collection does not move the position toward liquidation.
  - Only the protected account itself can produce this. A third party cannot cause it on someone else's account.

**Severity: Low. Impact: "Contract fails to deliver promised returns, but doesn't lose value".**

### Recommendation

In `OrderLiquidationBatchLogic.updateMarkPrice`, give the trigger bounty the same self-payment branch the Router already uses for order bounties:

```solidity
if (refundRecipient == plan.account) {
    // activate() has already taken the ProtectionTrigger bounty record, as the helper requires.
    IMarginClearinghouse(engine.clearinghouse())
        .releaseReservedExecutionBountyToSource(plan.account, plan.triggerBountyAssets);
} else {
    engine.creditBounty(
        plan.account, refundRecipient, plan.triggerBountyAssets, snapshot.markPrice, snapshot.publishTime
    );
}
```

`PositionProtectionBook.activate` already takes the `ProtectionTrigger` bounty record (`_recordActivation`, `PositionProtectionBook.sol` L556). This is the same precondition that `commitProtectedClose` relies on. The trader's carry then stays pending: it is assessed against the retained ceilings when the close executes and is recorded in its receipt, exactly as for a third-party trigger.

Add a regression test in which the protected account triggers its own protection with accrued carry above its ceilings, and assert that the generated close fails the ceiling, as the third-party control does.

## References

- Trigger appends `msg.sender` as the bounty recipient: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L370-L378
- Trigger path: activate, register and queue the child, then `creditBounty`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L583-L622
- Atomic cancel-and-close releases the trigger bounty without a checkpoint: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L167-L181
- Retained ceilings copied into the generated close: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L655-L680
- Child queued with a zero bounty and no commitment: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L436-L471
- Order bounty self-payment branch: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L314-L344
- `creditBounty` and the beneficiary checkpoint: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L460-L492 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L203-L217
- Carry realization: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1407-L1449
- No-checkpoint self-release helper: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L716-L732 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IMarginClearinghouse.sol#L371-L380
- Book takes the trigger bounty record on activation: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L547-L561
- Commitment effects included in close assessment: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluator.sol#L64-L67 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L474-L487
- Receipt copies the stored commitment and the execution assessment: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L703-L718 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L781-L799
- Self-execution vs external execution bounty handling: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L231-L235
- Protection limits "including commitment charges", trigger reverts if exceeded: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L276-L286
- Protection actions table: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L257-L267
- Receipt verification steps: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L298-L312
- Assessment carry includes commitment-time carry: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderV3Types.sol#L190-L213
- Close commitment carry included in bounds and totals: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L933-L945
- Protection trigger/retry carry assessed at execution: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L948-L960
- Carry collection rules for credits: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L690-L726
- Owner atomic exit bounded by close limits: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L535-L546
- Protection trigger carry and close limits: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L112-L126
- Users acting through public entrypoints, same-account fallback: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L400-L447
- AA client smart-account trigger builder: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa-client/src/keeperExecution.ts#L52-L55
- Engine unit test of `creditBounty(A, A, ...)`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/carry/CarryCreditCheckpoint.t.sol#L275-L289
- Known issue MED-9 (Cecuro AI audit, "Keepers can retry protection closes without the trader's financial limits") and its "Addressed" resolution.


## Proof of Concept

## Proof of Concept

A single Foundry test file runs against the unmodified audit commit (`a82d4fefb02768e5b3bddcc97dee371eee24fea8`) using the repository's own `BasePerpTest` harness.

### Setup

- **Release risk parameters**, asserted in `setUp`:
  - VPI 0.01e18 and max skew 0.4e18;
  - maintenance/initial/FAD margin 10/20/300 bps;
  - `baseCarryBps` 500, execution fee 4 bps, frozen spread 50 bps;
  - confidence ratio 10 bps with a 2,500 multiplier;
  - Senior cap 40M USDG / 80%;
  - protection trigger bounty 0.20 USDG.
- **Public paths only.**
  - The harness funds a 1,000,000 USDG Junior pool through the public asynchronous ERC-7540 deposit flow. The trader deposits through `MarginClearinghouse.deposit`.
  - The position is opened with the production V3 `OrderRouter.commitOrder(OrderRequest)` and `executeOrder`. Marks are installed through the public `updateMarkPrice`.
  - Protection is created with `PositionProtectionBook.createPositionProtection` and triggered with `triggerPositionProtection`. The generated close is executed by an unrelated keeper EOA through `executeOrder`.
- **Mocks.** MockPyth only supplies signed-price fixture data, and Pyth is out of scope. Only external EOAs are pranked (trader, keeper, mark updater); no call is pranked as a protocol contract. The harness Router is `LegacyOrderRouterHarness is OrderRouter`. That subclass only adds a legacy scalar `commitOrder` overload, which this PoC does not use.
- **Position.** A 300,000 USDG LONG with 20,000 USDG margin, about 15x. Take-profit at 0.90 and stop-loss at 1.10, with retained ceilings `maxGrossAccountDebitAssets = maxActionChargeAssets = 10 USDG`. Explicit fees and notional are unlimited. Time starts on Monday 2024-03-11 10:00 UTC and advances by whole days, so the market stays live.

### Tests

| Test | What it shows |
| --- | --- |
| `test_SelfTrigger_CollectsCarryOutsideRetainedCloseLimitsAndReceipt` | **The bug.** After 28 days, the trader triggers their own protection. Triggering does not revert and collects 301.753575 USDG of carry: one `CarryRealized` event for the trader, settlement 99,430.698080 -> 99,128.944505 USDG, pool +301.753575. The queued child holds the 10 USDG ceilings and a zero commitment. A keeper executes it. The verified receipt shows gross debit 124 atoms, carry 124 atoms, commitment carry 0. The protection ends `Executed` and the trader is flat. |
| `test_Control_ThirdPartyTrigger_SameLimitRejectsSameCarry` | **Control 1, same path.** The same protection, price, carry and ceilings, triggered by a keeper. No trader carry is collected at trigger, and the trader pays only the 0.20 USDG bounty. The child fails with `ConstraintViolation` / `GrossAccountDebit` (actual 301.753700, limit 10 USDG). Carry and pool roll back. The protection becomes `Latched` and the position stays open. |
| `test_Control_OwnerAtomicCancelAndClose_SameLimitRejectsSameCarry` | **Control 2, owner exit.** `cancelPositionProtectionAndCommitClose` with the same 10 USDG ceiling reverts with the exact error `OrderLifecycleBook__CommitmentBoundExceeded(GrossAccountDebit, 301753575, 10000000)`. Nothing changes and the protection stays `Armed`. |
| `test_ThreeDaysCarry_SelfTriggerExecutesWhileThirdPartyTriggerFails` | **Shorter horizon.** After three days, self-trigger collects 32.330851 USDG and the close executes (receipt gross debit 124 atoms). From the same snapshot, a keeper trigger's close fails `GrossAccountDebit` at 32.330976 against 10 USDG. |

### How to run

Save the file below as `packages/perps/test/poc/ProtectionSelfTriggerCarryBypassesCloseLimits.t.sol` in the audit repository, then run from the repository root:

```bash
forge test --root packages/perps --match-path test/poc/ProtectionSelfTriggerCarryBypassesCloseLimits.t.sol -vv
```

### Output

The test was run twice. Both runs produced identical results; only the wall-clock timing line differs.

Run 1:

```text
Ran 4 tests for test/poc/ProtectionSelfTriggerCarryBypassesCloseLimits.t.sol:ProtectionSelfTriggerCarryBypassesCloseLimitsTest
[PASS] test_Control_OwnerAtomicCancelAndClose_SameLimitRejectsSameCarry() (gas: 4132172)
Logs:
  owner atomic close rejected: gross debit actual: 301753575
  owner atomic close rejected: gross debit limit: 10000000

[PASS] test_Control_ThirdPartyTrigger_SameLimitRejectsSameCarry() (gas: 5116286)
Logs:
  third-party child gross debit actual: 301753700
  third-party child gross debit limit: 10000000
  trader settlement after failed child: 99430498080

[PASS] test_SelfTrigger_CollectsCarryOutsideRetainedCloseLimitsAndReceipt() (gas: 5575364)
Logs:
  retained close max gross debit / max action charge: 10000000
  carry collected inside self-trigger tx: 301753575
  trader settlement before trigger: 99430698080
  trader settlement after trigger: 99128944505
  pool assets before trigger: 1002449101920
  pool assets after trigger: 1002750855495
  child commitment carryCollectedAssets: 0
  receipt grossAccountDebitAssets: 124
  receipt actionChargeAssessedAssets: 0
  receipt carryAssets: 124
  receipt executionFeeAssets: 108000000
  receipt commitment carryCollectedAssets: 0

[PASS] test_ThreeDaysCarry_SelfTriggerExecutesWhileThirdPartyTriggerFails() (gas: 7296084)
Logs:
  three days: carry collected inside self-trigger tx: 32330851
  three days: self-triggered receipt gross debit: 124
  three days: keeper-triggered gross debit actual: 32330976
  three days: keeper-triggered gross debit limit: 10000000

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 10.01ms (15.20ms CPU time)

Ran 1 test suite in 10.49ms (10.01ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

Run 2:

```text
Ran 4 tests for test/poc/ProtectionSelfTriggerCarryBypassesCloseLimits.t.sol:ProtectionSelfTriggerCarryBypassesCloseLimitsTest
[PASS] test_Control_OwnerAtomicCancelAndClose_SameLimitRejectsSameCarry() (gas: 4132172)
Logs:
  owner atomic close rejected: gross debit actual: 301753575
  owner atomic close rejected: gross debit limit: 10000000

[PASS] test_Control_ThirdPartyTrigger_SameLimitRejectsSameCarry() (gas: 5116286)
Logs:
  third-party child gross debit actual: 301753700
  third-party child gross debit limit: 10000000
  trader settlement after failed child: 99430498080

[PASS] test_SelfTrigger_CollectsCarryOutsideRetainedCloseLimitsAndReceipt() (gas: 5575364)
Logs:
  retained close max gross debit / max action charge: 10000000
  carry collected inside self-trigger tx: 301753575
  trader settlement before trigger: 99430698080
  trader settlement after trigger: 99128944505
  pool assets before trigger: 1002449101920
  pool assets after trigger: 1002750855495
  child commitment carryCollectedAssets: 0
  receipt grossAccountDebitAssets: 124
  receipt actionChargeAssessedAssets: 0
  receipt carryAssets: 124
  receipt executionFeeAssets: 108000000
  receipt commitment carryCollectedAssets: 0

[PASS] test_ThreeDaysCarry_SelfTriggerExecutesWhileThirdPartyTriggerFails() (gas: 7296084)
Logs:
  three days: carry collected inside self-trigger tx: 32330851
  three days: self-triggered receipt gross debit: 124
  three days: keeper-triggered gross debit actual: 32330976
  three days: keeper-triggered gross debit limit: 10000000

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 10.68ms (15.07ms CPU time)

Ran 1 test suite in 11.37ms (10.68ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

### Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice A protected trader who triggers their own take-profit/stop-loss is paid the trigger bounty through
///         `CfdEngine.creditBounty(account, account, ...)`. That call checkpoints and collects the trader's accrued
///         carry after the generated close has been queued with a zero commitment, so the trader-retained close
///         limits and the close's authenticated receipt never see the debit. The same carry under the same limit is
///         rejected when a third party triggers the same protection, and when the owner uses the atomic
///         cancel-and-close path.
/// @dev Public entrypoints only. MockPyth is the only replaced dependency. Release risk parameters are asserted.
contract ProtectionSelfTriggerCarryBypassesCloseLimitsTest is BasePerpTest {

    address internal constant TRADER = address(0x7A0E);
    address internal constant KEEPER = address(0x4EE9E7);
    address internal constant MARK_UPDATER = address(0xB007);

    uint256 internal constant MARK = 1e8;
    uint256 internal constant TRIGGER_MARK = 90_000_000; // crosses the take-profit leg
    uint256 internal constant STOP_LOSS_MARK = 110_000_000;
    uint256 internal constant SIZE = 300_000e18;
    uint256 internal constant MARGIN = 20_000e6;
    /// @dev Trader-retained gross-account-debit and action-charge ceilings for the protection close (10 USDG).
    uint256 internal constant CLOSE_DEBIT_LIMIT = 10e6;

    bytes32 internal constant CARRY_REALIZED_TOPIC =
        keccak256("CarryRealized(address,uint256,uint256,uint256,uint256)");

    IPositionProtectionBook internal protectionBook;

    struct CarryTotals {
        uint256 realized;
        uint256 remaining;
        uint256 count;
    }

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
        _setSeniorCapacity(40_000_000e6, 8000);

        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.basketMaxConfidenceRatioBps = 10;
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);

        // Monday 2024-03-11 10:00 UTC, after configuration activation delays.
        vm.warp(1_710_151_200);
        vm.roll(vm.getBlockNumber() + 1);
        _installPublicMark(MARK);
        protectionBook = router.positionProtectionBook();

        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen spread");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "release confidence ratio");
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500, "release confidence multiplier");
        (uint256 vpi, uint256 skew, uint256 maintenance, uint256 initial, uint256 fad, uint256 carry,,,,) =
            engine.riskParams();
        assertEq(vpi, 0.01e18, "release VPI");
        assertEq(skew, 0.4e18, "release max skew");
        assertEq(maintenance, 10, "release maintenance margin");
        assertEq(initial, 20, "release initial margin");
        assertEq(fad, 300, "release FAD margin");
        assertEq(carry, 500, "release carry");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release senior cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release senior share");
    }

    /// @notice Bug: the trader's own trigger debits 301.753575 USDG of carry against a 10 USDG retained ceiling.
    ///         Triggering does not revert, the generated close then executes, and its verified receipt reports
    ///         only the 124 atoms of carry accrued between trigger and execution.
    function test_SelfTrigger_CollectsCarryOutsideRetainedCloseLimitsAndReceipt() public {
        (uint64 protectionId, bytes[] memory triggerData, uint256 expectedCarry) = _armedProtectionWithCarry(28 days);

        uint256 traderBalanceBefore = clearinghouse.balanceAssets(TRADER);
        uint256 poolBefore = pool.totalAssets();

        vm.recordLogs();
        vm.prank(TRADER);
        uint64 childOrderId = protectionBook.triggerPositionProtection(protectionId, triggerData);
        CarryTotals memory triggerCarry = _carryFor(vm.getRecordedLogs(), TRADER);

        assertEq(triggerCarry.count, 1, "self-trigger bounty credit checkpoints the protected trader");
        assertEq(triggerCarry.realized, expectedCarry, "all accrued carry is collected inside the trigger");
        assertEq(triggerCarry.remaining, 0, "carry is fully funded; no arrears remain");
        assertGt(triggerCarry.realized, CLOSE_DEBIT_LIMIT, "trigger-time debit exceeds the retained ceiling");
        assertEq(
            traderBalanceBefore - clearinghouse.balanceAssets(TRADER),
            triggerCarry.realized,
            "the bounty self-transfer nets out; carry is the trader's only balance loss"
        );
        assertEq(pool.totalAssets() - poolBefore, triggerCarry.realized, "collected carry reaches the pool");

        OrderV3Types.PendingIntent memory child = router.lifecycleBook().pendingIntent(childOrderId);
        assertEq(child.bounds.maxGrossAccountDebitAssets, CLOSE_DEBIT_LIMIT, "child carries retained gross ceiling");
        assertEq(child.bounds.maxActionChargeAssets, CLOSE_DEBIT_LIMIT, "child carries retained action ceiling");
        assertEq(child.commitment.carryCollectedAssets, 0, "child commitment omits trigger-time carry");
        assertEq(child.commitment.settlementBeforeAssets, 0, "child has no commitment custody snapshot");

        emit log_named_uint("retained close max gross debit / max action charge", CLOSE_DEBIT_LIMIT);
        emit log_named_uint("carry collected inside self-trigger tx", triggerCarry.realized);
        emit log_named_uint("trader settlement before trigger", traderBalanceBefore);
        emit log_named_uint("trader settlement after trigger", clearinghouse.balanceAssets(TRADER));
        emit log_named_uint("pool assets before trigger", poolBefore);
        emit log_named_uint("pool assets after trigger", pool.totalAssets());
        emit log_named_uint("child commitment carryCollectedAssets", child.commitment.carryCollectedAssets);

        bytes[] memory executionData = _mockPythUpdateData(TRIGGER_MARK);
        _startRecordingLogs();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(childOrderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "generated close executes");

        OrderV3Types.OrderReceipt memory receipt = _verifiedReceipt(router.lifecycleBook(), childOrderId);
        assertLe(receipt.economics.grossAccountDebitAssets, CLOSE_DEBIT_LIMIT, "receipt is within the ceiling");
        assertLe(receipt.economics.actionChargeAssessedAssets, CLOSE_DEBIT_LIMIT, "receipt is within the ceiling");
        assertLt(uint256(receipt.economics.carryAssets), 1e6, "receipt carry excludes the trigger-time debit");
        assertEq(receipt.commitment.carryCollectedAssets, 0, "verified receipt keeps the zero commitment");
        assertEq(
            uint8(protectionBook.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Executed),
            "protection completes"
        );
        _assertFlat(TRADER);

        emit log_named_uint("receipt grossAccountDebitAssets", receipt.economics.grossAccountDebitAssets);
        emit log_named_uint("receipt actionChargeAssessedAssets", receipt.economics.actionChargeAssessedAssets);
        emit log_named_uint("receipt carryAssets", uint256(receipt.economics.carryAssets));
        emit log_named_uint("receipt executionFeeAssets", receipt.economics.executionFeeAssets);
        emit log_named_uint("receipt commitment carryCollectedAssets", receipt.commitment.carryCollectedAssets);
    }

    /// @notice Control 1: the identical protection, price, carry and limits, triggered by a third party. The trader's
    ///         carry is left for the generated close, whose retained 10 USDG ceiling rejects it; protection latches.
    function test_Control_ThirdPartyTrigger_SameLimitRejectsSameCarry() public {
        (uint64 protectionId, bytes[] memory triggerData,) = _armedProtectionWithCarry(28 days);
        uint256 traderBalanceBefore = clearinghouse.balanceAssets(TRADER);
        uint256 poolBefore = pool.totalAssets();

        vm.recordLogs();
        vm.prank(KEEPER);
        uint64 childOrderId = protectionBook.triggerPositionProtection(protectionId, triggerData);
        CarryTotals memory triggerCarry = _carryFor(vm.getRecordedLogs(), TRADER);
        assertEq(triggerCarry.count, 0, "third-party trigger does not checkpoint the protected trader");
        assertEq(pool.totalAssets(), poolBefore, "third-party trigger collects no trader carry");
        assertGt(_expectedIndexedCarryAssets(TRADER), CLOSE_DEBIT_LIMIT, "deferred carry exceeds the ceiling");

        bytes[] memory executionData = _mockPythUpdateData(TRIGGER_MARK);
        _startRecordingLogs();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(childOrderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "bounded close fails");
        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "failure is a financial-constraint violation"
        );
        OrderV3Types.OrderReceipt memory receipt = _verifiedReceipt(router.lifecycleBook(), childOrderId);
        assertEq(
            uint8(receipt.failure.constraint),
            uint8(OrderV3Types.ConstraintKind.GrossAccountDebit),
            "retained gross-debit ceiling rejects the carry"
        );
        assertEq(receipt.failure.limit, CLOSE_DEBIT_LIMIT, "rejection uses the retained ceiling");
        assertGt(receipt.failure.actual, CLOSE_DEBIT_LIMIT, "carry is counted in the close's gross debit");
        assertEq(
            clearinghouse.balanceAssets(TRADER),
            traderBalanceBefore - router.positionProtectionTriggerBountyAssets(),
            "trader paid only the 0.20 USDG trigger bounty to the keeper; failed close rolls back carry"
        );
        assertEq(pool.totalAssets(), poolBefore, "failed close rolls back pool inflow");
        assertEq(
            uint8(protectionBook.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "protection latches for owner amendment"
        );
        (uint256 sizeAfter,,,,,,) = engine.positions(TRADER);
        assertEq(sizeAfter, SIZE, "position stays open");

        emit log_named_uint("third-party child gross debit actual", receipt.failure.actual);
        emit log_named_uint("third-party child gross debit limit", receipt.failure.limit);
        emit log_named_uint("trader settlement after failed child", clearinghouse.balanceAssets(TRADER));
    }

    /// @notice Control 2: the owner's own exit through `cancelPositionProtectionAndCommitClose` with the same ceiling.
    ///         The trigger bounty is released without a carry checkpoint and the canonical close commitment rejects
    ///         the same carry atomically.
    function test_Control_OwnerAtomicCancelAndClose_SameLimitRejectsSameCarry() public {
        (uint64 protectionId,, uint256 expectedCarry) = _armedProtectionWithCarry(28 days);
        uint256 traderBalanceBefore = clearinghouse.balanceAssets(TRADER);
        uint256 poolBefore = pool.totalAssets();

        OrderV3Types.OrderRequest memory boundedClose =
            _request(keccak256("BOUNDED_CLOSE"), CfdTypes.Side.LONG, SIZE, 0, true, CLOSE_DEBIT_LIMIT);
        vm.expectRevert(
            abi.encodeWithSelector(
                IOrderLifecycleBook.OrderLifecycleBook__CommitmentBoundExceeded.selector,
                OrderV3Types.ConstraintKind.GrossAccountDebit,
                expectedCarry,
                CLOSE_DEBIT_LIMIT
            )
        );
        vm.prank(TRADER);
        protectionBook.cancelPositionProtectionAndCommitClose(protectionId, boundedClose);

        assertEq(clearinghouse.balanceAssets(TRADER), traderBalanceBefore, "rejected commitment rolls back carry");
        assertEq(pool.totalAssets(), poolBefore, "rejected commitment rolls back pool inflow");
        assertEq(
            uint8(protectionBook.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "protection stays armed"
        );
        emit log_named_uint("owner atomic close rejected: gross debit actual", expectedCarry);
        emit log_named_uint("owner atomic close rejected: gross debit limit", CLOSE_DEBIT_LIMIT);
    }

    /// @notice Shorter horizon: three days of carry on the same position. Self-trigger still collects it outside the
    ///         ceiling and the close executes; the same protection triggered by a third party fails the ceiling.
    function test_ThreeDaysCarry_SelfTriggerExecutesWhileThirdPartyTriggerFails() public {
        (uint64 protectionId, bytes[] memory triggerData, uint256 expectedCarry) = _armedProtectionWithCarry(3 days);
        assertGt(expectedCarry, CLOSE_DEBIT_LIMIT, "three days of carry exceed the ceiling");
        uint256 branch = vm.snapshotState();

        vm.recordLogs();
        vm.prank(TRADER);
        uint64 selfChild = protectionBook.triggerPositionProtection(protectionId, triggerData);
        CarryTotals memory selfCarry = _carryFor(vm.getRecordedLogs(), TRADER);
        assertEq(selfCarry.realized, expectedCarry, "self-trigger collects the three days of carry");
        OrderV3Types.ExecutionResult memory selfResult = _executeAsKeeper(selfChild);
        assertEq(uint8(selfResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "self-triggered close runs");
        OrderV3Types.OrderReceipt memory selfReceipt = _verifiedReceipt(router.lifecycleBook(), selfChild);

        assertTrue(vm.revertToState(branch), "restore the pre-trigger state");
        vm.prank(KEEPER);
        uint64 keeperChild = protectionBook.triggerPositionProtection(protectionId, triggerData);
        OrderV3Types.ExecutionResult memory keeperResult = _executeAsKeeper(keeperChild);
        assertEq(
            uint8(keeperResult.status), uint8(OrderV3Types.LifecycleStatus.Failed), "keeper-triggered close fails"
        );
        OrderV3Types.OrderReceipt memory keeperReceipt = _verifiedReceipt(router.lifecycleBook(), keeperChild);
        assertEq(
            uint8(keeperReceipt.failure.constraint),
            uint8(OrderV3Types.ConstraintKind.GrossAccountDebit),
            "keeper-triggered close hits the gross ceiling"
        );

        emit log_named_uint("three days: carry collected inside self-trigger tx", selfCarry.realized);
        emit log_named_uint(
            "three days: self-triggered receipt gross debit", selfReceipt.economics.grossAccountDebitAssets
        );
        emit log_named_uint("three days: keeper-triggered gross debit actual", keeperReceipt.failure.actual);
        emit log_named_uint("three days: keeper-triggered gross debit limit", keeperReceipt.failure.limit);
    }

    // ---------------------------------------------------------------------------------------------------------
    // Helpers
    // ---------------------------------------------------------------------------------------------------------

    /// @dev Opens a 300,000-unit LONG with 20,000 USDG margin (about 15x), arms a take-profit at 0.90 with the
    ///      trader's 10 USDG gross-debit and action-charge ceilings, waits `carryPeriod`, and prepares a crossing
    ///      price update.
    function _armedProtectionWithCarry(
        uint256 carryPeriod
    ) internal returns (uint64 protectionId, bytes[] memory triggerData, uint256 expectedCarry) {
        _fundTrader(TRADER, 100_000e6);
        _openThroughRouter(TRADER, SIZE, MARGIN);
        _installPublicMark(MARK);

        vm.prank(TRADER);
        protectionId = protectionBook.createPositionProtection(_protectionParams());
        assertEq(router.positionProtectionTriggerBountyAssets(), 200_000, "release trigger bounty (0.20 USDG)");
        PositionProtectionTypes.CloseFinancialLimits memory limits = protectionBook.closeFinancialLimits(protectionId);
        assertEq(limits.maxGrossAccountDebitAssets, CLOSE_DEBIT_LIMIT, "retained gross ceiling");
        assertEq(limits.maxActionChargeAssets, CLOSE_DEBIT_LIMIT, "retained action ceiling");

        vm.warp(vm.getBlockTimestamp() + carryPeriod);
        triggerData = _mockPythUpdateData(TRIGGER_MARK);
        expectedCarry = _expectedIndexedCarryAssets(TRADER);
        assertGt(expectedCarry, CLOSE_DEBIT_LIMIT, "accrued carry exceeds the retained ceiling");
    }

    function _executeAsKeeper(
        uint64 orderId
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory executionData = _mockPythUpdateData(TRIGGER_MARK);
        _startRecordingLogs();
        vm.prank(KEEPER);
        result = router.executeOrder(orderId, executionData);
    }

    function _assertFlat(
        address account
    ) internal view {
        (uint256 size,,,,,,) = engine.positions(account);
        assertEq(size, 0, "position closed");
    }

    function _installPublicMark(
        uint256 price
    ) internal {
        bytes[] memory updateData = _mockPythUpdateData(price);
        vm.prank(MARK_UPDATER);
        router.updateMarkPrice(updateData);
    }

    function _openThroughRouter(address account, uint256 size, uint256 margin) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request =
            _request(keccak256("OPEN"), CfdTypes.Side.LONG, size, margin, false, type(uint256).max);
        vm.prank(account);
        orderId = router.commitOrder(request);

        bytes[] memory updateData = _mockPythUpdateData(MARK);
        vm.prank(MARK_UPDATER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public open executes");
    }

    function _request(
        bytes32 clientId,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 maxGrossDebit
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientId;
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = MARK;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: maxGrossDebit,
            maxActionChargeAssets: isClose ? maxGrossDebit : type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _protectionParams()
        internal
        pure
        returns (PositionProtectionTypes.PositionProtectionParams memory params)
    {
        params.takeProfitTriggerPrice = TRIGGER_MARK;
        params.stopLossTriggerPrice = STOP_LOSS_MARK;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: CLOSE_DEBIT_LIMIT,
            maxActionChargeAssets: CLOSE_DEBIT_LIMIT,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _carryFor(Vm.Log[] memory logs, address account) internal view returns (CarryTotals memory totals) {
        bytes32 accountTopic = bytes32(uint256(uint160(account)));
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter != address(engine) || logs[i].topics.length < 2
                    || logs[i].topics[0] != CARRY_REALIZED_TOPIC || logs[i].topics[1] != accountTopic
            ) {
                continue;
            }
            (uint256 realized,,, uint256 remaining) = abi.decode(logs[i].data, (uint256, uint256, uint256, uint256));
            totals.realized += realized;
            totals.remaining = remaining;
            ++totals.count;
        }
    }

}
```

