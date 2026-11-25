# \#99862 [SC-Low] Trigger-bounty carry checkpoint lets an owner-triggered protected close exceed its stored charge ceilings

**Submitted on Nov 15th 2026 at 14:34:02 UTC by @songthuy307 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99862
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

A take-profit/stop-loss protection stores four close ceilings chosen by its owner. The documentation says they bound each protected close attempt, "including commitment charges". When the owner herself calls `triggerPositionProtection`, the trigger bounty is credited back to her through `CfdEngine.creditBounty`, which first checkpoints the beneficiary's carry. The carry accrued on her position is therefore collected inside the trigger transaction, after the close attempt has been queued, and it is neither bounded by the ceilings nor recorded for that attempt. The evaluator then sees only the charges of the close itself, so the close executes although trigger carry plus close charges exceed `maxActionChargeAssets` and `maxGrossAccountDebitAssets`. In the PoC the same protection triggered by another address fails on the ceiling, and a 100 USDG action-charge ceiling does not stop a protected close for which 163.33 USDG is collected: 75.33 USDG of carry in the trigger transaction and 88.00 USDG at execution. The PoC shows this with the repository's test parameters and again with the release risk parameters. No value is lost, because the carry was owed. What fails is the documented ceiling. Only the owner's own trigger call has this effect; a third party cannot cause it.

Code reviewed at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (in-scope freeze). Paths are relative to `packages/perps/`.

## Vulnerability Details

**1. What the ceilings promise.**

- A protection "must explicitly supply" four close limits; each is "a `uint256` ceiling in 6-decimal USDG units for future triggered or retried close attempts" ([`CANONICAL_ENTRYPOINTS.md:121-126`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L121-L126)).
- `maxGrossAccountDebitAssets` is the "Maximum aggregate settlement debit, consumed trader claims, and reserved bounty", and `maxActionChargeAssets` is the "Maximum assessed action charge after carry, signed virtual price impact (VPI), fees, and frozen spread are combined" ([`WORKING_WITH_AI_AGENTS.md:102-103`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L102-L103)).
- "These limits apply per attempt, including commitment charges, not cumulatively across retries. If commitment would exceed a limit, triggering reverts and the owner can replace the `Armed` protection." ([`WORKING_WITH_AI_AGENTS.md:282-283`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L282-L283))
- The paragraph that defines the ceilings ends: "The separately reserved trigger bounty is paid when the protection fires; these ceilings apply to its generated close attempt." ([`CANONICAL_ENTRYPOINTS.md:125-126`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L125-L126)) It names the trigger bounty as the amount paid at trigger time outside the ceilings. It does not mention carry.

**2. A trigger by another account leaves the carry to be assessed with the close.**

The specification describes this transition directly. `ACCOUNTING_SPEC.md` lists "Protection trigger/retry" with the accounting "Queue through the protection path without ordinary commitment carry effects" and the note "carry is assessed at execution" ([`ACCOUNTING_SPEC.md:954`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L954)). The integration guide says: "A trigger or retry queues a new close without ordinary commitment carry effects; its execution still settles carry." ([`CANONICAL_ENTRYPOINTS.md:112-113`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L112-L113))

That is what happens when anyone other than the owner triggers. The trigger queues the close attempt with a zero bounty record and no commitment ([`OrderRouter.sol:440-472`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L440-L472)) and pays the trigger bounty to the caller. The owner's accrued carry stays pending. At execution the evaluator assesses it together with the close: carry is part of both `grossAccountDebitAssets` and `actionChargeAssessedAssets` ([`CfdOrderPolicyEvaluatorBase.sol:409-415`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L409-L415)). If the total exceeds a ceiling, the attempt fails with `ConstraintViolation` and the protection relatches.

**3. A trigger by the owner moves the carry outside the ceilings.**

- `triggerPositionProtection` forwards `msg.sender` as the bounty recipient ([`PositionProtectionBook.sol:371-379`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L371-L379)); the Router decodes it as `refundRecipient` ([`OrderLiquidationBatchLogic.sol:593-601`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L593-L601)).
- The trigger branch activates the protection, queues the attempt and only then calls `engine.creditBounty(plan.account, refundRecipient, ...)` ([`OrderLiquidationBatchLogic.sol:611-620`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L611-L620)).
- `creditBounty` calls `_checkpointBountyRecipient(beneficiary, ...)` before the transfer ([`CfdEngine.sol:471-493`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L471-L493)). If the beneficiary has a position, this realizes its accrued carry from settlement ([`CfdEngine.sol:203-217`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L203-L217)).
- When the beneficiary is `plan.account`, the owner's accrued carry is collected here, as far as her margin and free settlement cover it. The attempt's commitment keeps `carryCollectedAssets == 0`, and that field is the only commit-time carry the evaluator adds back ([`CfdOrderPolicyEvaluator.sol:65-67`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluator.sol#L65-L67), [`CfdOrderPolicyEvaluatorBase.sol:474-487`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L474-L487)).

The close is therefore assessed without the carry and executes. The carry was charged in the same transaction that created the attempt, but it is not counted against the ceilings stored for it. For an owner trigger, "carry is assessed at execution" does not hold: the trigger transaction itself collects her carry, and nothing is left for the execution to assess.

**4. Elsewhere the code base avoids this checkpoint when a bounty returns to its own source.**

`creditBounty` documents why it checkpoints: "For a beneficiary with an open position, carry is checkpointed before the settlement credit changes reachable collateral." ([`CfdEngine.sol:461-462`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L461-L462)) For a bounty that returns to the account that funded it, the clearinghouse has a separate function. Its NatSpec reads: "Releases a settled order or protection-trigger bounty back to the same account that funded it. [...] Unlike the general Engine bounty-credit path, this exact self-transfer changes only the reserved classification and deliberately skips carry checkpointing. [...] Atomic protection cancellation uses this before bounded close carry collection." ([`MarginClearinghouse.sol:716-732`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L716-L732))

The Router uses that function in three places where a bounty goes back to its source account:

- Terminal order settlement, when the bounty recipient is the order's own account ([`OrderRouter.sol:333-343`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L333-L343)).
- Recovery of an expired order's bounty ([`OrderRouter.sol:217-222`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L217-L222)).
- Cancel-and-close of an armed protection, where the comment gives the reason in terms of the close bounds: "Release it without a carry checkpoint so all commitment carry is collected, bounded and recorded by the ordinary close path below." ([`OrderLiquidationBatchLogic.sol:174-181`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L174-L181))

For the cancel-and-close path the same intent is stated three more times:

- `SECURITY.md`: "The Router releases the unused trigger bounty without checkpointing carry before ordinary close commitment. All commitment carry is then collected and recorded by the canonical close path, including its gross-debit and action-charge bounds" ([`SECURITY.md:535-539`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L535-L539)).
- The team's response to Cecuro MED-30: "commitment carry is collected and checked against the close limits" (audit report p. 69).
- The repository's tests `test_CancelAndCommitClose_AccruedCarryCannotBypassGrossDebitBound` and `test_CancelAndCommitClose_AccruedCarryCannotBypassActionChargeBound`. They assert that accrued carry is counted against the close's gross-debit and action-charge bounds when a protection is cancelled and closed in one call ([`PositionProtection.t.sol:1969-2007`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/trader/PositionProtection.t.sol#L1969-L2007)).

The trigger branch does not use it. When the trigger caller is the account that funded the protection-trigger bounty, the bounty goes through the general `creditBounty` path. A plain cancellation of an armed protection also checkpoints carry when it releases the bounty ([`MarginClearinghouse.sol:826-839`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L826-L839)), but it creates no close. The trigger does: the checkpoint happens inside the entrypoint that creates the bounded close, which is the kind of bypass those two tests exclude for cancel-and-close.

**5. Relation to earlier findings and documentation.**

- The ceilings were introduced by the fix for Cecuro MED-9 ("Keepers can retry protection closes without the trader's financial limits", audit report p. 27). The team's response says the protection "copies them into both the initial close and every permissionless retry". The copy is correct, and this report is not MED-9. It concerns the mechanism that fix introduced: for an owner-triggered attempt, carry collected by the trigger transaction stays outside the copied ceilings.
- The checkpoint on a bounty credit is itself documented and tested. `ACCOUNTING_SPEC.md` states that "keeper bounty credits first collect all available margin and free settlement toward carry" ([`ACCOUNTING_SPEC.md:702-705`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L702-L705)), `SECURITY.md` says the same for claim and bounty credits ([`SECURITY.md:954-957`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L954-L957)), and the Engine test `test_SelfBountyProtectsReserveThenReleasesFullCredit` calls `creditBounty` with the same account as source and beneficiary ([`CarryCreditCheckpoint.t.sol:275-289`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/carry/CarryCreditCheckpoint.t.sol#L275-L289)). None of these covers the interaction reported here. They do not say that carry collected this way is outside the ceilings of a close queued in the same transaction, and I found no repository test in which the protection owner is the trigger caller.
- An owner can also realize her carry deliberately before a close, for example with a zero-amount withdrawal; `SECURITY.md` mentions these "User-triggered checkpoints, including zero-amount withdrawals" ([`SECURITY.md:1423-1427`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1423-L1427)). That is a separate action she chooses. Here the checkpoint is a side effect of the trigger entrypoint itself, inside the transaction that creates the bounded attempt, which is the situation the cancel-and-close path was written to avoid.
- For transparency: my earlier report in this competition (confirmed, Low) covers carry collected at the commit of a public V3 open/increase, which escapes that order's signed bounds. This report is related but concerns a different defect. The entrypoint is `triggerPositionProtection`, not `commitOrder`. The sink is `CfdEngine.creditBounty` and `_checkpointBountyRecipient`, not the clearinghouse checkpoint taken when order margin is locked. The bypassed limits are the protection's stored close ceilings, not an order's request bounds. The fix is different, and fixing either one leaves the other in place.
- The behaviour is not listed in `SECURITY.md` "Known Limitations" or "Router and keeper limitations". The AI scan finding "Protection trigger bounty is paid before a zero-bounty close is executed" is about keeper incentives after the bounty is paid; it does not mention the carry checkpoint.

**Limits of the issue.**

- It occurs only when the trigger caller is the owner's own account, for example an owner who runs her own trigger bot so that she does not depend on third-party keepers, or to keep the trigger bounty. A third party cannot cause it.
- The carry is owed in any case, so nobody loses value. The effect is that the ceilings the owner stored do not bound what is collected from her for the protected close, and the close executes where a trigger by anyone else would have failed and left the decision to her.
- It affects `maxActionChargeAssets` and `maxGrossAccountDebitAssets`, the two ceilings that include carry. `maxGrossAccountDebitAssets` adds the carry in full. For `maxActionChargeAssets` the evaluator nets a VPI rebate against the carry ([`CfdOrderPolicyEvaluatorBase.sol:562-572`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L562-L572)), so a close that earns a VPI rebate at least as large as its carry is not affected on that ceiling.

## Impact Details

Selected impact: **Contract fails to deliver promised returns, but doesn't lose value** (Low).

The contract does not deliver a documented guarantee: the owner's stored ceilings are meant to bound each protected close attempt including the charges taken when it is created, and for an owner-triggered attempt they do not.

Numbers from the PoC with the repository's test risk parameters (Alice: LONG 200,000 USDG notional at 1.00, 60,000 USDG margin, stop-loss at 1.10, position held 28 days; accrued carry 75.33 USDG; execution fee of the close 88.00 USDG):

- `maxActionChargeAssets = 100 USDG`.
  - Trigger by a keeper: the evaluator assesses 163.33 USDG (carry 75.33 + fee 88.00) against the ceiling and rejects the attempt with `ConstraintViolation(ActionCharge, 163.328469, 100.000000)`. The protection relatches and the position stays open.
  - Trigger by Alice: the trigger transaction debits 75.33 USDG of carry. The evaluator then assesses 88.00 USDG and the close executes. Collected in total across the trigger and the close: 163.33 USDG, 63.33 USDG above her ceiling.
- `maxGrossAccountDebitAssets = 20,100 USDG`.
  - Trigger by a keeper: 20,163.33 USDG assessed (price loss 20,000 + carry 75.33 + fee 88.00), rejected.
  - Trigger by Alice: 20,088.00 USDG assessed, executed. Total across the trigger and the close: 20,163.33 USDG.

The PoC repeats the four tests with the release risk parameters from `DEPLOYMENT.md` ([`DEPLOYMENT.md:128-151`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L128-L151)): VPI factor 0.01e18, maintenance margin 10 bps, initial margin 20 bps. The execution fee and the trigger bounty already have their release values in the fixture. In that run a second trader holds a larger SHORT, so that Alice's close is assessed a VPI charge and not a rebate. Accrued carry is 75.23 USDG:

- `maxActionChargeAssets = 600 USDG`. A keeper trigger is rejected at 646.21 USDG. Alice's trigger debits 75.23 USDG of carry, the evaluator assesses 570.97 USDG and the close executes. Total across the trigger and the close: 646.21 USDG.
- `maxGrossAccountDebitAssets = 20,600 USDG`. A keeper trigger is rejected at 20,646.21 USDG. After Alice's trigger the evaluator assesses 20,570.97 USDG and the close executes. Total across the trigger and the close: 20,646.21 USDG.

In both parameter sets the amount outside the ceiling is the carry accrued since the position's last checkpoint and collected by the trigger transaction, so it grows with position size, utilization and holding time.

## References

- Trigger path: [`PositionProtectionBook.sol:371-379`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L371-L379), [`OrderLiquidationBatchLogic.sol:593-601`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L593-L601), [`OrderLiquidationBatchLogic.sol:611-620`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L611-L620), [`OrderRouter.sol:440-472`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L440-L472).
- Carry checkpoint on bounty credit: [`CfdEngine.sol:461-462`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L461-L462), [`CfdEngine.sol:471-493`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L471-L493), [`CfdEngine.sol:203-217`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L203-L217).
- Evaluator: [`CfdOrderPolicyEvaluator.sol:65-67`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluator.sol#L65-L67), [`CfdOrderPolicyEvaluatorBase.sol:409-415`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L409-L415), [`CfdOrderPolicyEvaluatorBase.sol:474-487`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L474-L487), [`CfdOrderPolicyEvaluatorBase.sol:562-572`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L562-L572).
- Own-account bounty release without a checkpoint: [`MarginClearinghouse.sol:716-732`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L716-L732), [`OrderRouter.sol:333-343`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L333-L343), [`OrderRouter.sol:217-222`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L217-L222), [`OrderLiquidationBatchLogic.sol:174-181`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L174-L181).
- Release parameters: [`DEPLOYMENT.md:128-151`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L128-L151).
- Documentation: [`CANONICAL_ENTRYPOINTS.md:121-126`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L121-L126), [`WORKING_WITH_AI_AGENTS.md:102-103`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L102-L103), [`WORKING_WITH_AI_AGENTS.md:282-283`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L282-L283), [`CANONICAL_ENTRYPOINTS.md:125-126`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L125-L126), [`SECURITY.md:535-539`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L535-L539), [`SECURITY.md:954-957`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L954-L957), [`SECURITY.md:1423-1427`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1423-L1427), [`ACCOUNTING_SPEC.md:954`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L954), [`CANONICAL_ENTRYPOINTS.md:112-113`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L112-L113), [`ACCOUNTING_SPEC.md:702-705`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L702-L705).
- Repository tests: the cancel-and-close bound tests [`PositionProtection.t.sol:1969-2007`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/trader/PositionProtection.t.sol#L1969-L2007); the Engine self-bounty test [`CarryCreditCheckpoint.t.sol:275-289`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/carry/CarryCreditCheckpoint.t.sol#L275-L289).
- Cecuro AI audit, MED-9 and the team response: [`audits/plether-ai-audit-report-2026-10-26.pdf`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf), pp. 27-28; MED-30 team response, p. 69.

## Recommendation

Either of the following closes the gap:

- In the trigger branch, when `refundRecipient == plan.account`, return the trigger bounty with `releaseReservedExecutionBountyToSource` (no carry checkpoint), as terminal order settlement and `commitProtectedClose` already do. The carry then stays pending and is assessed against the ceilings at execution.
- Alternatively, record the carry collected by the trigger transaction in the attempt's commitment so that `_includeCommitment` counts it, and revert the trigger when it exceeds a ceiling, as the documentation describes.


## Proof of Concept

Runs in the repository's own Foundry suite at the in-scope commit. The PoC contract inherits the repository's `PositionProtectionTest` (`packages/perps/test/perps/spec/trader/PositionProtection.t.sol`) to reuse its fixtures and helpers. `BasePerpTest` deploys the in-scope contracts unmodified, with mocked Pyth updates and mock USDG only, which the rules allow because both are out of scope. Its Router is the repository's `LegacyOrderRouterHarness`, a test-only subclass of the production `OrderRouter` that adds a legacy `commitOrder` overload; the PoC does not call it. The fixture helpers `_fundTrader` and `_open` fund Alice and create her position (`_open` is the repository helper that books the position directly through the Engine). Everything after that uses production entrypoints only: `createPositionProtection`, `triggerPositionProtection`, `executeOrder`, and the public view `CfdOrderPolicyEvaluator.assessCommittedOrder`.

The file contains two contracts. `ProtectionOwnerTriggerCarryPocTest` runs on the repository's test risk parameters (zero VPI factor). `ProtectionOwnerTriggerCarryReleaseParamsPocTest` repeats the same four tests with the release risk parameters from `packages/perps/DEPLOYMENT.md` (VPI factor 0.01e18, maintenance margin 10 bps, initial margin 20 bps; the 4 bps execution fee and the 0.20 USDG trigger bounty already have their release values in the fixture). With a non-zero VPI factor the evaluator nets a VPI rebate against carry in the assessed action charge, so in the release contract Bob `0xB0B` holds a larger SHORT first and Alice's close is assessed a VPI charge instead of a rebate. In each contract the two tests of a pair differ only in who calls the trigger.

Save the file below as `packages/perps/test/perps/spec/trader/ProtectionOwnerTriggerCarryPoc.t.sol` and run:

```bash
cd packages/perps
forge test --match-path test/perps/spec/trader/ProtectionOwnerTriggerCarryPoc.t.sol --match-test "test_(Control|Poc)_" -vv
```

`--match-test` selects the four PoC tests of each contract, because both contracts also inherit the repository's own protection tests.

Setup of the first contract (test parameters): Alice `0xA11CE` opens LONG 200,000 USDG notional at 1.00 with 60,000 USDG margin and arms a stop-loss at 1.10 with one finite close ceiling. The position is held for 28 days, during which 75.33 USDG of carry accrues. The market then trades at 1.10; the protected close has an 88.00 USDG execution fee and a 20,000 USDG price loss.

1. `test_Control_ThirdPartyTriggerFailsOnActionChargeCeiling` (`maxActionChargeAssets = 100 USDG`): the keeper `0x7106` triggers. Alice pays only the reserved 0.20 USDG trigger bounty and her carry stays pending. The production evaluator rejects the attempt with `ConstraintViolation(ActionCharge, 163.328469, 100.000000)`; the test reads these values from the evaluator's revert data and also checks that the amount without the carry would be inside the ceiling. Then `executeOrder` terminalizes it as `Failed / ConstraintViolation`, the protection is `Latched` and the position is still open.
2. `test_Poc_OwnerTriggerExecutesAboveActionChargeCeiling` (same ceiling): Alice triggers. The trigger transaction debits 75.33 USDG of carry from her account, the attempt records `carryCollectedAssets == 0`, the evaluator assesses 88.00 USDG, and `executeOrder` executes the close. Trigger carry plus close charge is 163.33 USDG against a 100 USDG ceiling.
3. `test_Control_ThirdPartyTriggerFailsOnGrossDebitCeiling` (`maxGrossAccountDebitAssets = 20,100 USDG`): as test 1; the evaluator rejects 20,163.33 USDG.
4. `test_Poc_OwnerTriggerExecutesAboveGrossDebitCeiling` (same ceiling): as test 2; the evaluator assesses 20,088.00 USDG and the close executes. The total is 20,163.33 USDG.

The release-parameter contract runs the same four tests with Bob's SHORT of 300,000 USDG notional opened first, and with ceilings of 600 USDG (`maxActionChargeAssets`) and 20,600 USDG (`maxGrossAccountDebitAssets`). Accrued carry is 75.23 USDG. The keeper-triggered attempts are rejected at 646.21 USDG and 20,646.21 USDG. After Alice's own trigger the evaluator assesses 570.97 USDG and 20,570.97 USDG, and both closes execute.

Output:

```
Ran 4 tests for test/perps/spec/trader/ProtectionOwnerTriggerCarryPoc.t.sol:ProtectionOwnerTriggerCarryPocTest
[PASS] test_Control_ThirdPartyTriggerFailsOnActionChargeCeiling() (gas: 3132170)
Logs:
  third-party trigger: ceiling: 100.000000
  third-party trigger: carry counted at execution: 75.328469
  third-party trigger: amount assessed against the ceiling (rejected): 163.328469
[PASS] test_Control_ThirdPartyTriggerFailsOnGrossDebitCeiling() (gas: 3132988)
Logs:
  third-party trigger: ceiling: 20100.000000
  third-party trigger: carry counted at execution: 75.328469
  third-party trigger: amount assessed against the ceiling (rejected): 20163.328469
[PASS] test_Poc_OwnerTriggerExecutesAboveActionChargeCeiling() (gas: 3423200)
Logs:
  owner trigger: ceiling: 100.000000
  owner trigger: carry debited by the trigger transaction (not counted): 75.328469
  owner trigger: amount assessed against the ceiling (accepted): 88.000000
  owner trigger: total across the trigger and the close: 163.328469
[PASS] test_Poc_OwnerTriggerExecutesAboveGrossDebitCeiling() (gas: 3422372)
Logs:
  owner trigger: ceiling: 20100.000000
  owner trigger: carry debited by the trigger transaction (not counted): 75.328469
  owner trigger: amount assessed against the ceiling (accepted): 20088.000000
  owner trigger: total across the trigger and the close: 20163.328469
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 30.70ms (36.92ms CPU time)
Ran 4 tests for test/perps/spec/trader/ProtectionOwnerTriggerCarryPoc.t.sol:ProtectionOwnerTriggerCarryReleaseParamsPocTest
[PASS] test_Control_ThirdPartyTriggerFailsOnActionChargeCeiling() (gas: 4051416)
Logs:
  third-party trigger: ceiling: 600.000000
  third-party trigger: carry counted at execution: 75.231735
  third-party trigger: amount assessed against the ceiling (rejected): 646.205266
[PASS] test_Control_ThirdPartyTriggerFailsOnGrossDebitCeiling() (gas: 4052234)
Logs:
  third-party trigger: ceiling: 20600.000000
  third-party trigger: carry counted at execution: 75.231735
  third-party trigger: amount assessed against the ceiling (rejected): 20646.205266
[PASS] test_Poc_OwnerTriggerExecutesAboveActionChargeCeiling() (gas: 4199411)
Logs:
  owner trigger: ceiling: 600.000000
  owner trigger: carry debited by the trigger transaction (not counted): 75.231735
  owner trigger: amount assessed against the ceiling (accepted): 570.973531
  owner trigger: total across the trigger and the close: 646.205266
[PASS] test_Poc_OwnerTriggerExecutesAboveGrossDebitCeiling() (gas: 4198583)
Logs:
  owner trigger: ceiling: 20600.000000
  owner trigger: carry debited by the trigger transaction (not counted): 75.231735
  owner trigger: amount assessed against the ceiling (accepted): 20570.973531
  owner trigger: total across the trigger and the close: 20646.205266
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 37.16ms (49.90ms CPU time)
Ran 2 test suites in 39.54ms (67.86ms CPU time): 8 tests passed, 0 failed, 0 skipped (8 total tests)
```

PoC file:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdOrderPolicyEvaluator} from "@plether/perps/interfaces/ICfdOrderPolicyEvaluator.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

/// @notice PoC: when the protection owner is the trigger caller, the trigger-bounty credit checkpoints the owner's
///         accrued carry after the close attempt has been queued. That carry is neither bounded nor recorded for the
///         attempt, so the protected close executes above the owner's stored `maxActionChargeAssets` and
///         `maxGrossAccountDebitAssets`. The same protection triggered by any other caller fails on the ceiling.
///         Reuses the repository's own PositionProtection fixtures and helpers. This contract runs on the
///         repository's test risk parameters; the contract at the end of the file repeats the four tests with the
///         release risk parameters from packages/perps/DEPLOYMENT.md.
contract ProtectionOwnerTriggerCarryPocTest is PositionProtectionTest {

    /// @dev Alice: LONG 200,000 USDG notional at 1.00 with 60,000 USDG margin, stop-loss at 1.10, held 28 days.
    uint256 internal constant SIZE = 20 * POSITION_SIZE;
    uint256 internal constant MARGIN_ASSETS = 60_000e6;
    uint256 internal constant HOLD = 28 days;

    /// @dev Ceilings chosen by Alice. Each sits above the charges of the close itself and below close + accrued carry.
    function _actionChargeCeiling() internal pure virtual returns (uint256) {
        return 100e6;
    }

    function _grossDebitCeiling() internal pure virtual returns (uint256) {
        return 20_100e6;
    }

    function _limits(
        uint256 maxGrossAccountDebitAssets,
        uint256 maxActionChargeAssets
    ) internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: maxGrossAccountDebitAssets,
            maxActionChargeAssets: maxActionChargeAssets,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    /// @dev Positions of other traders that exist before Alice opens. None in this contract.
    function _openOtherTraders() internal virtual {}

    function _armAndHold(
        PositionProtectionTypes.CloseFinancialLimits memory limits
    ) internal returns (uint64 protectionId) {
        _openOtherTraders();
        _fundTrader(ALICE, 100_000e6);
        _open(ALICE, CfdTypes.Side.LONG, SIZE, MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        params.closeLimits = limits;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
        vm.warp(vm.getBlockTimestamp() + HOLD);
    }

    function _triggerAs(
        address caller,
        uint64 protectionId
    ) internal returns (uint64 linkedOrderId) {
        _mockPythUpdateData(LONG_STOP_LOSS);
        bytes[] memory updateData = _livePriceData(LONG_STOP_LOSS);
        vm.prank(caller);
        linkedOrderId = protectionActions.triggerPositionProtection(protectionId, updateData);
    }

    function _status(
        uint64 protectionId
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return protectionViews.getPositionProtection(protectionId).status;
    }

    function _execute(
        uint64 orderId
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory executionData = _mockPythUpdateData(LONG_STOP_LOSS);
        vm.prank(EXECUTION_KEEPER);
        result = router.executeOrder(orderId, executionData);
    }

    /// @dev Decodes `CfdOrderPolicyEvaluator__ConstraintViolation(kind, assessed, limit)` from revert data.
    function _decodeViolation(
        bytes memory reason
    ) internal pure returns (OrderV3Types.ConstraintKind kind, uint256 assessed, uint256 limit) {
        require(
            bytes4(reason) == ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector,
            "evaluator must reject with ConstraintViolation"
        );
        bytes memory args = new bytes(reason.length - 4);
        for (uint256 i = 0; i < args.length; i++) {
            args[i] = reason[i + 4];
        }
        (kind, assessed, limit) = abi.decode(args, (OrderV3Types.ConstraintKind, uint256, uint256));
    }

    /// @dev Any caller other than the owner triggers. The owner's carry stays pending, the production evaluator
    ///      counts it against her ceiling, the attempt fails with ConstraintViolation and the protection relatches.
    function _thirdPartyTriggerFails(
        PositionProtectionTypes.CloseFinancialLimits memory limits,
        OrderV3Types.ConstraintKind expectedKind,
        uint256 ceiling
    ) internal {
        uint64 protectionId = _armAndHold(limits);
        uint256 balanceBefore = clearinghouse.balanceAssets(ALICE);
        uint64 childOrderId = _triggerAs(TRIGGER_KEEPER, protectionId);

        uint256 pendingCarry = _expectedIndexedCarryAssets(ALICE);
        assertGt(pendingCarry, 0, "a third-party trigger does not collect the owner's carry");
        assertEq(
            balanceBefore - clearinghouse.balanceAssets(ALICE),
            router.positionProtectionTriggerBountyAssets(),
            "the owner only pays the reserved trigger bounty"
        );

        // The production evaluator rejects the attempt. Read the amount it assessed from its revert data.
        uint256 assessed;
        try policyEvaluator.assessCommittedOrder(address(engine), childOrderId, EXECUTION_KEEPER, LONG_STOP_LOSS) {
            fail();
        } catch (bytes memory reason) {
            (OrderV3Types.ConstraintKind kind, uint256 assessedAssets, uint256 limit) = _decodeViolation(reason);
            assertEq(uint8(kind), uint8(expectedKind), "violated constraint");
            assertEq(limit, ceiling, "limit reported by the evaluator is the owner's ceiling");
            assessed = assessedAssets;
        }
        assertGt(assessed, ceiling, "with the carry the attempt is above the ceiling");
        assertLe(assessed, ceiling + pendingCarry, "without the carry the attempt would be inside the ceiling");

        OrderV3Types.ExecutionResult memory result = _execute(childOrderId);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "attempt must fail");
        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "attempt must fail on the owner's ceiling"
        );
        assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, SIZE, "the ceiling kept the position open");

        emit log_named_decimal_uint("third-party trigger: ceiling", ceiling, 6);
        emit log_named_decimal_uint("third-party trigger: carry counted at execution", pendingCarry, 6);
        emit log_named_decimal_uint("third-party trigger: amount assessed against the ceiling (rejected)", assessed, 6);
    }

    /// @dev Alice triggers her own protection. The trigger-bounty credit collects her accrued carry in the trigger
    ///      transaction, after the attempt is queued. The attempt records no commitment carry, the evaluator sees only
    ///      the charges of the close itself, and the close executes although the total exceeds her ceiling.
    function _ownerTriggerExecutes(
        PositionProtectionTypes.CloseFinancialLimits memory limits,
        bool grossDebit,
        uint256 ceiling
    ) internal {
        uint64 protectionId = _armAndHold(limits);
        uint256 accruedCarry = _expectedIndexedCarryAssets(ALICE);
        uint256 balanceBefore = clearinghouse.balanceAssets(ALICE);
        uint64 childOrderId = _triggerAs(ALICE, protectionId);

        uint256 carryDebitedByTrigger = balanceBefore - clearinghouse.balanceAssets(ALICE);
        assertGe(carryDebitedByTrigger, accruedCarry, "the trigger transaction debits the accrued carry");
        assertEq(_expectedIndexedCarryAssets(ALICE), 0, "no carry is left for the attempt to count");
        CfdEnginePlanTypes.CloseCommitment memory commitment =
            router.lifecycleBook().pendingIntent(childOrderId).commitment;
        assertEq(commitment.carryCollectedAssets, 0, "the attempt records no commitment carry");

        OrderV3Types.ExecutionAssessment memory assessment =
            policyEvaluator.assessCommittedOrder(address(engine), childOrderId, EXECUTION_KEEPER, LONG_STOP_LOSS);
        uint256 assessed = grossDebit ? assessment.grossAccountDebitAssets : assessment.actionChargeAssessedAssets;
        assertEq(assessment.carryAssets, 0, "the evaluator sees no carry");
        assertLe(assessed, ceiling, "the evaluator sees an amount inside the ceiling");

        OrderV3Types.ExecutionResult memory result = _execute(childOrderId);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "the close executes");
        assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Executed));
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "the position is closed");

        uint256 total = carryDebitedByTrigger + assessed;
        assertGt(total, ceiling, "trigger carry + close exceeds the ceiling stored for this close");

        emit log_named_decimal_uint("owner trigger: ceiling", ceiling, 6);
        emit log_named_decimal_uint(
            "owner trigger: carry debited by the trigger transaction (not counted)", carryDebitedByTrigger, 6
        );
        emit log_named_decimal_uint("owner trigger: amount assessed against the ceiling (accepted)", assessed, 6);
        emit log_named_decimal_uint("owner trigger: total across the trigger and the close", total, 6);
    }

    function test_Control_ThirdPartyTriggerFailsOnActionChargeCeiling() public {
        _thirdPartyTriggerFails(
            _limits(type(uint256).max, _actionChargeCeiling()),
            OrderV3Types.ConstraintKind.ActionCharge,
            _actionChargeCeiling()
        );
    }

    function test_Poc_OwnerTriggerExecutesAboveActionChargeCeiling() public {
        _ownerTriggerExecutes(_limits(type(uint256).max, _actionChargeCeiling()), false, _actionChargeCeiling());
    }

    function test_Control_ThirdPartyTriggerFailsOnGrossDebitCeiling() public {
        _thirdPartyTriggerFails(
            _limits(_grossDebitCeiling(), type(uint256).max),
            OrderV3Types.ConstraintKind.GrossAccountDebit,
            _grossDebitCeiling()
        );
    }

    function test_Poc_OwnerTriggerExecutesAboveGrossDebitCeiling() public {
        _ownerTriggerExecutes(_limits(_grossDebitCeiling(), type(uint256).max), true, _grossDebitCeiling());
    }

}

/// @notice The same four tests with the release risk parameters from packages/perps/DEPLOYMENT.md
///         (vpiFactor 0.01e18, maintenance margin 10 bps, initial margin 20 bps, FAD margin 300 bps, carry 500 bps).
///         The execution fee (4 bps) and the trigger bounty (0.20 USDG) already equal the release values in the
///         fixture. With a non-zero VPI factor a close can earn a VPI rebate, which the evaluator nets against carry
///         in the assessed action charge. Bob therefore holds a larger SHORT first, so that Alice's close is
///         assessed a VPI charge instead of a rebate and both ceilings can be shown.
contract ProtectionOwnerTriggerCarryReleaseParamsPocTest is ProtectionOwnerTriggerCarryPocTest {

    /// @dev Bob: SHORT 300,000 USDG notional at 1.00 with 150,000 USDG margin, opened before Alice.
    function _openOtherTraders() internal override {
        _fundTrader(BOB, 200_000e6);
        _open(BOB, CfdTypes.Side.SHORT, 30 * POSITION_SIZE, 150_000e6, MARK_PRICE);
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

    /// @dev Ceilings chosen by Alice. Each sits above the charges of the close itself and below close + accrued carry.
    function _actionChargeCeiling() internal pure override returns (uint256) {
        return 600e6;
    }

    function _grossDebitCeiling() internal pure override returns (uint256) {
        return 20_600e6;
    }

}
```

