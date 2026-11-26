# \#99469 [SC-Low] Owner-sent protection trigger collects the owner's pending carry via creditBounty, outside its close limits and the receipt

**Submitted on Nov 13th 2026 at 19:34:46 UTC by @Lester for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99469
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The documents say a protection trigger has no carry effect, and that carry is bounded and recorded with the close:

- "Protection trigger/retry | Queue through the protection path without ordinary commitment carry effects | Append a fresh authenticated zero record; carry is assessed at execution." (`ACCOUNTING_SPEC.md:954`)
- "A trigger or retry queues a new close without ordinary commitment carry effects; its execution still settles carry." (`CANONICAL_ENTRYPOINTS.md:112-113`)
- "Receipts and financial bounds include the direct collection once, in addition to the residual action settlement" (`ACCOUNTING_SPEC.md:723-724`)
- "These limits apply per attempt, including commitment charges, not cumulatively across retries." (`WORKING_WITH_AI_AGENTS.md:282`)

The code pays the trigger reward with `engine.creditBounty(plan.account, refundRecipient, ...)` (`router/OrderLiquidationBatchLogic.sol:617-619`), and `creditBounty` first collects the pending carry of the reward recipient (`CfdEngine.sol:487`, `:203-217`). The recipient is the caller of `triggerPositionProtection`. When the caller is the account that owns the protection, that account's whole pending carry is collected inside the trigger transaction. This collection is not checked against the protection's close limits and is not in the receipt of the close; the close that follows is assessed with only the carry of the seconds after the trigger.

- **Who pays:** the account that owns the protection ("the owner"), in its own trigger transaction.
- **Who can cause it:** only that account. Another account's trigger does not touch the owner's carry (PoC tests 2 and 5a).
- **Numbers (PoC, release parameters; LONG 1,000,000 USDG notional, stop-loss armed for 30 days):**
  - the owner's ceiling `maxActionChargeAssets` = 1,015.050281 USDG (the close charge quoted at arming, 914.050281, plus 101.00 headroom);
  - the owner triggers: 333.246575 USDG of carry is collected in the trigger; the close executes with a receipt charge of 914.033157; in total 1,247.279732 USDG of charges, **232.229451 above the ceiling**; the receipt shows carry 0.000257 and commitment carry 0;
  - another account triggers, same state, same ceiling: the close is refused with `ActionCharge`, actual 1,247.279732 > limit 1,015.050281, and the protection becomes `Latched`.
- **What does not change:** the total the owner is charged is the same on both paths (1,247.279732); the pool and the treasury receive the same amounts; the only transfer that differs is the 0.20 USDG trigger reward (tests 3 and 4).

## Vulnerability Details

**Root cause** (pinned commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`)

`router/OrderLiquidationBatchLogic.sol:611-620`, trigger branch of `updateMarkPrice`:
```solidity
if (isProtectionTrigger) {
    uint64 linkedOrderId = host.nextCommitId();
    IPositionProtectionBook.TriggerPlan memory plan =
        protectionBook.activate(protectionId, snapshot.markPrice, snapshot.publishTime, linkedOrderId);
    _registerProtectionTrigger(host, protectionId, linkedOrderId, plan);
    _queueProtectionAttempt(host, linkedOrderId, plan.account, plan.side, plan.size);
    engine.creditBounty(
        plan.account, refundRecipient, plan.triggerBountyAssets, snapshot.markPrice, snapshot.publishTime
    );
}
```
`CfdEngine.sol:482-492`, `creditBounty`:
```solidity
bool syncCurve = _positions[beneficiary].lots > 0;
if (syncCurve) { expectedOldHash = _beginTerminalCurveMutation(beneficiary); }
_checkpointBountyRecipient(beneficiary, price, publishTime);        // :203-217 -> _realizeCarryFromSettlement(beneficiary)
clearinghouse.transferReservedSettlement(sourceAccount, beneficiary, amountAssets);
```

- `refundRecipient` is the caller of `PositionProtectionBook.triggerPositionProtection`: the Book appends `msg.sender` to the Router call (`PositionProtectionBook.sol:375-376`) and the Router reads it back (`OrderLiquidationBatchLogic.sol:596-600`).
- There is no branch for `refundRecipient == plan.account`. With the owner as caller the call is `creditBounty(account, account, ...)`: the account's pending carry is collected from position margin and free settlement and sent to the pool (`CfdEngine.sol:1407-1441`), then the reward returns to the account it came from.
- The close attempt was registered two lines earlier by `_registerProtectionTrigger` (`:628-653`) with the four retained limits (`_protectionCloseRequest`, `:657-680`). That registration records no commitment carry (the receipt's `commitment.carryCollectedAssets` is 0), and the limits are checked only when the close executes, on the assessment of that moment.
- So the carry collected by the trigger is in no bound and in no receipt.

**The same situation on two neighbouring paths has a no-collection branch**

| path | code | document |
|---|---|---|
| order reward, executor = the order's own account | `OrderRouter.sol:336-337`: `if (bountyRecipient == order.account) { clearinghouse.releaseReservedExecutionBountyToSource(order.account, bountyAssets); } else { engine.creditBounty(...) }` | "Carry-safe reservation resolution \| Opening-bounty release does not checkpoint carry." (`SECURITY.md:384`) |
| owner exit of an armed protection (`cancelPositionProtectionAndCommitClose`): the unused trigger reward returns to the owner | `OrderLiquidationBatchLogic.sol:176-179`: "Release it without a carry checkpoint so all commitment carry is collected, bounded and recorded by the ordinary close path below." | "The Router releases the unused trigger bounty without checkpointing carry before ordinary close commitment. All commitment carry is then collected and recorded by the canonical close path, including its gross-debit and action-charge bounds" (`SECURITY.md:537-539`); project tests `test_CancelAndCommitClose_AccruedCarryCannotBypassGrossDebitBound`, `..ActionChargeBound` (`PositionProtection.t.sol:1969`, `:1989`) |
| protection trigger, caller = the owner | `OrderLiquidationBatchLogic.sol:617-619`: always `creditBounty` | "without ordinary commitment carry effects" (`ACCOUNTING_SPEC.md:954`) |

PoC test 5b runs the first row: an order's own account executes it, the reward is released, and the account's pending carry is unchanged.

**Relation to the audit item MED-9 (marked Addressed)**

MED-9, "Keepers can retry protection closes without the trader's financial limits": "The protection retains trader-selected ceilings for execution notional, gross account debit, action charges and explicit fees, and copies them into both the initial close and every permissionless retry." (team comment; fix `eadc5c58`, 2026-10-23). The item's impact text: "A protection close can consume arbitrarily more fees and gross account debit than the trader would be allowed to authorize through a normal bounded V3 order."

- On the path shown here two of the four retained ceilings do not bound what the close costs: `maxActionChargeAssets` and `maxGrossAccountDebitAssets` (PoC test 3: with either one set one atom below the amount the close needs, the close is refused after another account's trigger and executes after the owner's trigger). `maxExecutionNotionalAssets` and `maxExplicitFeesAssets` hold on both paths.
- The lines `OrderLiquidationBatchLogic.sol:611-620` are outside the fix diff (`git blame`: 2026-08-24/25, 2026-10-15 and 2026-10-27; no line from `eadc5c58`, and `git show eadc5c58` does not contain the `creditBounty` call).
- Stated plainly: MED-9's own precondition is "A keeper or other external caller triggers the protection". Here the caller is the owner. We do not claim that a keeper can exceed the ceilings.

The program rule for fixed items: "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug." (program page, Known Issues)

**Distinct from our own report #98198**

#98198 (confirmed) is about carry collected inside `commitOrder` of an **increase** order (`router/OrderReservationAccounting.sol:108-128`, `MarginClearinghouse.lockReservedSettlement`), which is outside that order's pinned bounds and commitment record. This report is about a different transaction and a different code path: the **trigger reward credit** of a protection (`OrderLiquidationBatchLogic.sol:617-619` -> `CfdEngine.creditBounty`), and the bounds and receipt of the **protected close**. Both rest on the same spec sentence (`ACCOUNTING_SPEC.md:723`).

Measured separation: we applied the first option of the fix recommended in #98198 to a local copy (commitment carry of opens recorded and checked in `recordCommitment`, `_includeCommitment` applied to opens; two files, `OrderCommitHandler.sol` and `CfdOrderPolicyEvaluator.sol`). On that copy the effect of #98198 is gone, and the PoC of this report gives the same result as on the pinned tree (see Proof of Concept). The protected close is registered by `_registerProtectionTrigger`, never through the order-commit path that the #98198 fix changes.

**Sequence (the owner's own transactions: one)**

1. The owner has a LONG position and arms a stop-loss with `createPositionProtection`, with `closeLimits.maxActionChargeAssets` set from the close quote of that moment plus headroom.
2. Time passes; carry accrues on the position. The mark reaches the stop.
3. The owner calls `triggerPositionProtection` itself (the function is open to any account).
   - Inside this transaction its pending carry is collected and the reward returns to it.
4. Any account executes the queued close. It is assessed with (almost) no carry, passes the owner's ceiling and executes.

The same sequence with another account as the caller in step 3: the close is refused on the owner's ceiling and the protection becomes `Latched`.

## Impact Details

**Measured effect** (PoC, USDG; release risk parameters; pool 5,000,000 Junior + 5,000,000 Senior; the owner LONG 10,000 lots = 1,000,000 notional at 1.0000 with 100,000 margin; another trader SHORT of the same size; stop at 1.0100; 30 days between arming and the stop)

| | the owner triggers | another account triggers |
|---|---|---|
| carry pending before the trigger | 333.246575 | 333.246575 |
| collected in the trigger transaction | 333.246575 (pool +333.246575) | 0 |
| trigger reward paid away by the owner | 0 (returns to it) | 0.20 |
| ceiling `maxActionChargeAssets` = 1,015.050281: close | **executes** | **refused**: `ActionCharge`, actual 1,247.279732 > 1,015.050281; `Latched`, position open |
| close receipt: action charge / carry / commitment carry | 914.033157 / 0.000257 / 0 | (no execution) |
| charges the owner pays in total | 1,247.279732 = 232.229451 above the ceiling | 0.20 (reward only) |

With all four limits unlimited (both closes execute; test 3):

| | the owner triggers | another account triggers |
|---|---|---|
| charges borne by the owner (carry + execution fee + price impact) | 1,247.279732 | 1,247.279732 |
| of which carry | 333.246832 (333.246575 in the trigger + 0.000257 in the close) | 333.246832 (in the close) |
| close receipt: action charge | 914.033157 | 1,247.279732 |
| close receipt: gross account debit | 10,914.033157 | 11,247.279732 |
| receipt short of what was paid | 333.246575 (both fields) | 0 |
| owner's end worth | 1,988,852.494912 | 1,988,852.294912 (0.20 less: the reward) |

Each of the four limits, set one atom below what the close needs after another account's trigger (test 3):

| limit | value set | another account triggers | the owner triggers |
|---|---|---|---|
| `maxExecutionNotionalAssets` | 1,009,999.999999 | refused (`ExecutionNotional`) | refused |
| `maxGrossAccountDebitAssets` | 11,247.279731 | refused (`GrossAccountDebit`) | **executes** |
| `maxActionChargeAssets` | 1,247.279731 | refused (`ActionCharge`) | **executes** |
| `maxExplicitFeesAssets` | 403.999999 | refused (`ExplicitFees`) | refused |

Every other party, from before the trigger to after the close, unlimited limits (test 4):

| party | the owner triggers | another account triggers |
|---|---|---|
| owner | -11,247.279732 | -11,247.479732 |
| pool cash (LP side) | +10,843.279732 | +10,843.279732 |
| treasury | +404.000000 | +404.000000 |
| the other account (keeper) | 0 | +0.200000 |
| sum | 0 | 0 |

**Preconditions and limits, stated plainly**

- **Only the owner can cause it.** At both `creditBounty` call sites the recipient is the caller: the trigger (`OrderLiquidationBatchLogic.sol:617`, caller of `triggerPositionProtection`) and order settlement (`OrderRouter.sol:339`, caller of `executeOrder`, and only when it is not the order's own account). No entry point takes a recipient parameter. Test 5a: another account's trigger leaves the owner's carry at 333.246575. Test 5b: at the order site neither the order's own account nor another executor causes a collection of the order owner's carry. The liquidation reward (`CfdEngine.sol:1336-1338`) is likewise paid to the caller and skips the liquidated account (read, not tested).
- **The owner pays the same total on both paths.** Carry is owed either way; the trigger moves when it is collected and whether the limits and the receipt see it. Nobody gains except that the owner keeps its own 0.20 reward.
- **No other party is worse off.** Pool and treasury receive the same amounts (table above).
- **A documented owner action reaches the same end state.** After another account's trigger the owner can make an ordinary deposit; "A deposit credits custody and checkpoints carry" (`CANONICAL_ENTRYPOINTS.md:31`). Test 7: a 1 USDG deposit collects the 333.246575 and the close then executes under the same ceiling with the same receipt charge 914.033157. So the close limits do not bound carry that the owner pays through its own other transactions. What this report shows is narrower: the trigger transaction itself collects it, where the documents say the trigger has no carry effect, and the two neighbouring paths were given a no-collection branch.
- **Project test of the mechanism.** `test/perps/accounting/carry/CarryCreditCheckpoint.t.sol:275` (`test_SelfBountyProtectsReserveThenReleasesFullCredit`, 2026-09-09) calls `engine.creditBounty(ACCOUNT, ACCOUNT, ...)` directly and expects the carry collection. It predates the no-collection branch of the order path (2026-10-15) and the spec row quoted first (2026-10-26); it does not name the trigger, the close limits or the receipt.
- **Size.** The amount outside the limits is the carry accrued since the account's last carry collection: 333.246575 over 30 days here (about 11.1 per day for this position and pool). The close executes instead of being refused only when the ceiling lies between the charge without that carry and the charge with it.
- The book in the PoC is balanced by another trader's SHORT so that the close has a positive price-impact charge; that trader takes no part in the sequence.
- Not done: a control with a fix installed.

**Impact rows of the program, element by element**

| program row (verbatim) | elements | met by a test? |
|---|---|---|
| critical \| Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield | a party ends with funds that belonged to another | **not met**: owner's charges equal on both paths (`test_3`), pool / treasury equal (`test_4`) |
| critical \| Permanent freezing of funds | funds that cannot be moved | **not met**: the owner withdraws its whole balance (`test_6`) |
| critical \| Protocol insolvency | liabilities above assets | **not met**: closed system, pool receives the same (`test_4`) |
| high \| Theft of unclaimed yield | yield of LPs goes elsewhere or is not paid | **not met**: the same carry reaches the pool on both paths, 333.246832 (`test_3`, `test_4`) |
| high \| Permanent freezing of unclaimed yield | same, frozen | **not met** (`test_4`) |
| high \| Temporary freezing of funds for at least 24 hours | funds unavailable, duration | **not met** (`test_6`) |
| medium \| Temporary freezing of funds for at least 1 hour | funds unavailable, duration | **not met** (`test_6`) |
| medium \| Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol) | (a) an account other than the damaged party causes it; (b) damage to users or the protocol | (a) **not met**: `test_5a`, `test_5b` (another account cannot cause the collection); (b) **not met**: `test_3`, `test_4` (same charges, same receipts of pool and treasury) |
| medium \| Smart contract unable to operate due to lack of token funds | a contract short of tokens | **not met** (`test_4`) |
| medium \| Theft of gas | - | not applicable, no test |
| medium \| Block stuffing | - | not applicable, no test |
| medium \| Unbounded gas consumption | - | not applicable, no test |
| low \| Contract fails to deliver promised returns, but doesn't lose value | (a) a documented promise is not delivered; (b) no value is lost | (a) **met**: `test_1` (carry 333.246575 collected in the trigger; 232.229451 above the owner's ceiling; receipt carry 0.000257, commitment carry 0), `test_2` (same ceiling refuses the close after another account's trigger), `test_3` (two of the four limits; receipt short by 333.246575); (b) **met**: `test_4`, `test_6` (sum of all changes 0) |

**Impact (Low):** "Contract fails to deliver promised returns, but doesn't lose value". It is the highest row whose every element has an assertion. The row one level above (Griefing) was examined with tests 5a, 5b and 7: both of its elements are unmet, not one. There is no row below Low in the program; the floor is an Insight.

- **Bound:** only the owner's own trigger; amount = the carry accrued since its last collection (333.246575 in the PoC); the owner's total cost is unchanged.
- **Likelihood:** the owner triggering its own protection is foreseen ("allows another operator or the trader to act when the usual automation is unavailable", docs.plether.com, Keepers and permissionless execution); it needs carry accrued since the last collection, which every open position has.

## Known-issue review

Sources searched (2026-11-13): the repository documents `packages/perps/*.md` and NatSpec at the pinned commit; the project's own tests; `git log` of the repository (1,496 commits; message search and `-S` on the two call sites); the five audit texts linked by the program (Cecuro AI audit 2026-10-26, AI scan of 2026-10-15, pre-audit report, collaborative audit 2026-05-12, Cantina 2026-01); the program page's Known Issues; public GitHub lists of `Plether-Fi/plether-core` (issues, pull requests, commits since the pin) and `immunefi-team/audit-comp-plether` (issues disabled, no pull requests); docs.plether.com (index and the pages "Take profit and stop loss", "Keepers and permissionless execution", "Trading costs: fees, carry and VPI").

Closest items and why they differ:

| item | what it says | difference |
|---|---|---|
| Cecuro MED-9 (Addressed) | a keeper's trigger or retry closes the trader without financial limits | fixed for the close attempt; the reward credit of the trigger is outside the fix; here the caller is the owner |
| Cecuro MED-30 (Addressed, partly open: no automatic frozen-market trigger) | manual exit `cancelPositionProtectionAndCommitClose`; "commitment carry is collected and checked against the close limits" | the sibling path that got the no-collection release; the trigger path did not |
| Cecuro LOW-34 (Addressed by documentation) | the evaluator's carry field can overstate collected carry on a close | opposite direction (over-report), ordinary close |
| AI scan, Low "Protection trigger bounty is paid before a zero-bounty close is executed" | the caller is paid at trigger time and may not finish the close | same three code lines, a different claim (incentive); nothing about carry or the owner as caller |
| `ACCOUNTING_SPEC.md:702-705`, `SECURITY.md:954-957`, NatSpec `CfdEngine.sol:461-463` | "claim payments and keeper bounty credits first collect all available margin and free settlement toward carry" | the documented rule for a reward recipient; it does not say the reward of a trigger sent by the protected account collects that account's carry outside its close limits |
| project test `CarryCreditCheckpoint.t.sol:275` | Engine-level `creditBounty(ACCOUNT, ACCOUNT)` collects carry | see "Preconditions and limits" |
| `WORKING_WITH_AI_AGENTS.md:345` | "An order receipt alone does not contain that full history or the separate trigger bounty movement." | about the reward movement, not a carry collection |
| Program Known Issues: MED-7, MED-13, MED-16, LOW-57, MED-22 (carry rounding), MED-24, MED-29, MED-30, HIGH-1 | - | none concerns the trigger reward or carry at a trigger |
| plether-core issue #137; pull requests #90, #115, #124; commits after the pin (#136) | deployed risk configuration; carry collection before credits; zero close bounties; MED-9 / MED-30 fix; test speed | no change of the trigger reward credit; no public fix |
| our own #98198 | carry at increase commitment | different path; separation measured (above) |

## Recommended fix

In the trigger branch, release a reward that returns to its own source without a carry collection, as the two neighbouring paths do:
```solidity
if (refundRecipient == plan.account) {
    clearinghouse.releaseReservedExecutionBountyToSource(plan.account, plan.triggerBountyAssets);
} else {
    engine.creditBounty(plan.account, refundRecipient, plan.triggerBountyAssets, snapshot.markPrice, snapshot.publishTime);
}
```
Alternative: keep the collection and record it in the close attempt's commitment (`carryCollectedAssets`, custody before and after) so that the retained limits and the receipt include it once.

## References

- Code (pinned commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`): `router/OrderLiquidationBatchLogic.sol#L585-L620`, `#L628-L680`, `#L168-L181`; `CfdEngine.sol#L203-L217`, `#L460-L493`, `#L1407-L1441`; `OrderRouter.sol#L314-L344`; `PositionProtectionBook.sol#L371-L379`.
- Documents: `ACCOUNTING_SPEC.md` :702-705, :723-724, :954; `CANONICAL_ENTRYPOINTS.md` :31, :112-113; `SECURITY.md` :384, :537-539, :954-957; `WORKING_WITH_AI_AGENTS.md` :231, :276-284, :343-345.
- Project tests: `test/perps/spec/trader/PositionProtection.t.sol` :1969, :1989; `test/perps/accounting/carry/CarryCreditCheckpoint.t.sol` :275.
- Audit: Cecuro AI audit 2026-10-26, MED-9, MED-30, LOW-34; AI scan 2026-10-15, "Protection trigger bounty is paid before a zero-bounty close is executed".

## Proof of Concept

## Proof of Concept

Runnable Foundry test in the repository's own suite at the pinned commit. No in-scope contract is modified; the only mocks are the fixture's MockPyth and USDG mock (both outside the scope). No `vm.store`, no `vm.etch`, no protocol address is impersonated. Release risk parameters (`DEPLOYMENT.md`: carry 500 bps, trigger reward 0.20 USDG).

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the test below as packages/perps/test/PocS7_SelfTriggerCarry.t.sol
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/PocS7_*' -vv   # Foundry 1.5.1
```
No env vars or RPC needed. Expected: `8 passed; 0 failed`. Sparse mode compiles only what the test needs (194 files, about 90 s cold).

**Accounts and every `vm.prank` in the file**

| address in the test | role | calls made under `vm.prank` |
|---|---|---|
| `ACCOUNT` | the trader that owns the position and its protection ("the owner"); no privileged role | `commitOrder`, `createPositionProtection`, `triggerPositionProtection` (tests 1-3, 6), `clearinghouse.deposit` (test 7), `clearinghouse.withdraw` (test 6) |
| `KEEPER` | "another account"; no privileged role | `updateMarkPrice`, `executeOrder`, `triggerPositionProtection` (controls), `pool.settleLpEpoch` (setup; permissionless) |
| `ACCOUNT_2` | another trader; no privileged role | `commitOrder`, `executeOrder` (test 5b) |
| `JLP`, `SLP` | liquidity providers; no privileged role | `requestDeposit`, `claimDeposit` (setup) |
| the test contract | deployer / owner of the fixture | only in `setUp`: the fixture's deployment and `_setSeniorCapacity` (release Senior cap 40M / 80 %: `proposePoolConfig`, `finalizePoolConfig`). No owner call after `setUp` |

**Setup (USDG; prices with 8 decimals)**
1. Pool: 5,000,000 Junior + 5,000,000 Senior, deposited through the vaults' request / settle / claim flow.
2. `ACCOUNT_2` opens SHORT 10,000 lots, then `ACCOUNT` opens LONG 10,000 lots (1,000,000 notional, 100,000 margin) at 1.0000; both through `commitOrder` + `executeOrder`.

**Steps of the leading sequence (test 1)**
1. `ACCOUNT` reads the close charge at the stop price in the current book (914.050281, of which execution fee 404.00) and arms a stop-loss at 1.0100 with `maxActionChargeAssets` = 1,015.050281 (quote + 25 % of the fee); the other three limits are unlimited.
2. 30 days pass. Pending carry of `ACCOUNT`: 333.246575.
3. The mark is 1.0100. `ACCOUNT` calls `triggerPositionProtection`. In this transaction: `ACCOUNT` -333.246575, pool +333.246575, pending carry 0.
4. `KEEPER` executes the queued close two seconds later: Executed. Receipt: action charge 914.033157, carry 0.000257, commitment carry 0.
5. Carry collected in the trigger + receipt charge = 1,247.279732, which is 232.229451 above the ceiling.

**Control (test 2):** the same state and ceiling, `KEEPER` calls `triggerPositionProtection`: `ACCOUNT` pays 0.20, its carry stays 333.246575, the close is refused (`ActionCharge`, actual 1,247.279732, limit 1,015.050281), the protection is `Latched`, the position stays open.

**Other tests:** 3 totals on both paths with unlimited limits, and each of the four limits one atom below the need; 4 every other party and the sum of changes; 5a another account's trigger cannot collect the owner's carry; 5b at the order call site a reward that returns to the order's own account is released without a carry collection, and another executor does not collect the order owner's carry either; 6 the elements of the impact rows (the promise, no value lost, nothing frozen: the owner withdraws its whole balance); 7 control for the stated limit: after another account's trigger, a 1 USDG deposit by the owner collects the carry and the close executes under the same ceiling.

**The same file on a tree with the recommended fix of our report #98198 applied** (first option of that report: commitment carry of opens recorded and bounded; two files changed): `8 passed; 0 failed`, and every logged amount is equal to the run on the pinned tree. The two reports have different causes.

Output of the run (clean copy of the pinned tree; compiler warnings left out, none of them names the test file; amounts are USDG atoms, 6 decimals):
```
Ran 8 tests for test/PocS7_SelfTriggerCarry.t.sol:PocS7_SelfTriggerCarryTest
[PASS] test_1_OwnerTriggersOwnProtection_CarryCollectedOutsideCloseLimitsAndReceipt() (gas: 5527577)
Logs:
    owner's ceiling: close charge quoted at arming / execution fee in it / maxActionChargeAssets 914050281 404000000 1015050281
    [owner triggers] carry due before the trigger / carry left right after it 333246575 0
    [owner triggers] trigger transaction: change of account / pool cash / keeper
  -333246575
  333246575
  0
    [owner triggers] close status (2 = Executed, 3 = Failed) / protection status / position lots 2 4 0
    [owner triggers] close receipt: action charge assessed / collected / gross account debit 914033157 914033157 10914033157
    [owner triggers] close receipt: execution fee / carry / commitment carry 404000000 257 0
    [owner triggers] close receipt: price-impact charge (signed) / realized price PnL (signed)
  510032900
  -10000000000
    [owner triggers] account worth: before the trigger / after the close attempt 2000099774644 1988852494912
    [owner triggers] close receipt: failed constraint (4 = ActionCharge) / actual / limit 0 0 0
    carry collected in the trigger + close charge in the receipt = paid / above the ceiling by 1247279732 232229451

[PASS] test_2_Control_AnotherAccountTriggers_SameCeilingRefusesTheClose() (gas: 8566015)
Logs:
    [another account triggers] carry due before the trigger / carry left right after it 333246575 333246575
    [another account triggers] trigger transaction: change of account / pool cash / keeper
  -200000
  0
  200000
    [another account triggers] close status (2 = Executed, 3 = Failed) / protection status / position lots 3 8 10000
    [another account triggers] close receipt: action charge assessed / collected / gross account debit 0 0 0
    [another account triggers] close receipt: execution fee / carry / commitment carry 0 0 0
    [another account triggers] close receipt: price-impact charge (signed) / realized price PnL (signed)
  0
  0
    [another account triggers] account worth: before the trigger / after the close attempt 2000099774644 2000099574644
    [another account triggers] close receipt: failed constraint (4 = ActionCharge) / actual / limit 4 1247279732 1015050281
    same ceiling: close after another account's trigger / after the owner's trigger (2 = Executed, 3 = Failed) 3 2
    assessed charge refused on the other account's path / carry + charge paid on the owner's path 1247279732 1247279732

[PASS] test_3_Totals_BothPaths_AndEachOfTheFourLimits() (gas: 32930686)
Logs:
  TOTALS with unlimited ceilings (both closes execute)
    [owner triggers] carry due before the trigger / carry left right after it 333246575 0
    [owner triggers] trigger transaction: change of account / pool cash / keeper
  -333246575
  333246575
  0
    [owner triggers] close status (2 = Executed, 3 = Failed) / protection status / position lots 2 4 0
    [owner triggers] close receipt: action charge assessed / collected / gross account debit 914033157 914033157 10914033157
    [owner triggers] close receipt: execution fee / carry / commitment carry 404000000 257 0
    [owner triggers] close receipt: price-impact charge (signed) / realized price PnL (signed)
  510032900
  -10000000000
    [owner triggers] account worth: before the trigger / after the close attempt 2000099774644 1988852494912
    [owner triggers] close receipt: failed constraint (4 = ActionCharge) / actual / limit 0 0 0
    [another account triggers] carry due before the trigger / carry left right after it 333246575 333246575
    [another account triggers] trigger transaction: change of account / pool cash / keeper
  -200000
  0
  200000
    [another account triggers] close status (2 = Executed, 3 = Failed) / protection status / position lots 2 4 0
    [another account triggers] close receipt: action charge assessed / collected / gross account debit 1247279732 1247279732 11247279732
    [another account triggers] close receipt: execution fee / carry / commitment carry 404000000 333246832 0
    [another account triggers] close receipt: price-impact charge (signed) / realized price PnL (signed)
  510032900
  -10000000000
    [another account triggers] account worth: before the trigger / after the close attempt 2000099774644 1988852294912
    [another account triggers] close receipt: failed constraint (4 = ActionCharge) / actual / limit 0 0 0
    charges borne by the account (carry + fee + price impact): owner path / other path 1247279732 1247279732
    carry paid: owner path (in trigger + in close) / other path (in close) 333246832 333246832
    account end worth: owner path / other path 1988852494912 1988852294912
    trigger reward paid away: owner path / other path 0 200000
    owner path, receipt short by: action charge / gross account debit 333246575 333246575
  EACH LIMIT one atom below the other-account path's value: constraint that refuses there / owner-path close status
    limit index (0 notional, 1 gross debit, 2 action charge, 3 explicit fees) / limit 0 1009999999999
      other-account path: status / failed constraint; owner path: status 3 2 3
    limit index (0 notional, 1 gross debit, 2 action charge, 3 explicit fees) / limit 1 11247279731
      other-account path: status / failed constraint; owner path: status 3 3 2
    limit index (0 notional, 1 gross debit, 2 action charge, 3 explicit fees) / limit 2 1247279731
      other-account path: status / failed constraint; owner path: status 3 4 2
    limit index (0 notional, 1 gross debit, 2 action charge, 3 explicit fees) / limit 3 403999999
      other-account path: status / failed constraint; owner path: status 3 5 3

[PASS] test_4_OtherParties_PoolTreasuryKeeper_AndConservation() (gas: 6921981)
Logs:
  change from before the trigger to after the close (unlimited ceilings): owner path, then other-account path
    account
  -11247279732
  -11247479732
    pool cash (LP side)
  10843279732
  10843279732
    treasury
  404000000
  404000000
    keeper (the other account)
  0
  200000
    pool totalAssets
  10843279732
  10843279732

[PASS] test_5a_TriggerSite_RecipientIsTheCaller_AnotherAccountCannotCollectTheOwnersCarry() (gas: 1890347)
Logs:
    another account triggers: owner's carry due before / after 333246575 333246575

[PASS] test_5b_OrderSite_OwnRewardIsReleasedWithoutCarryCollection() (gas: 2480494)
Logs:
    order site, own execution: carry due before / after / reward released 3999 3999 200000
    order site, another executor: carry due before / after / reward paid 3999 3999 200000

[PASS] test_6_ImpactRowElements() (gas: 8599996)
Logs:
    owner path under its own ceiling: paid above the ceiling by / account withdrew 232229451 1988852494912

[PASS] test_7_Control_OwnerDepositAfterAnotherAccountsTrigger_SameEndState() (gas: 5514549)
Logs:
    other account triggers, owner deposits 1 USDG, close: status / carry collected by the deposit / receipt charge 2 333246575 914033157

Suite result: ok. 8 passed; 0 failed; 0 skipped; finished in 54.94ms (95.74ms CPU time)

Ran 1 test suite in 146.12ms (54.94ms CPU time): 8 tests passed, 0 failed, 0 skipped (8 total tests)
```

### Full test code (`packages/perps/test/PocS7_SelfTriggerCarry.t.sol`)
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Position protection: the trigger pays its reward with engine.creditBounty(protected account, trigger caller, ...).
// creditBounty first collects the pending carry of the reward RECIPIENT. When the protected account triggers its own
// protection it is the recipient, so its whole pending carry is collected inside the trigger transaction. That
// collection is not checked against the protection's close limits and is not in the close's receipt; the close that
// follows is assessed with (almost) no carry.
//
//   [DOC-1] packages/perps/ACCOUNTING_SPEC.md:954  "Protection trigger/retry | Queue through the protection path
//           without ordinary commitment carry effects | Append a fresh authenticated zero record; carry is assessed
//           at execution."
//   [DOC-2] packages/perps/ACCOUNTING_SPEC.md:723-724  "Receipts and financial bounds include the direct collection
//           once, in addition to the residual action settlement"
//   [DOC-3] packages/perps/WORKING_WITH_AI_AGENTS.md:282  "These limits apply per attempt, including commitment
//           charges, not cumulatively across retries."
//   [DOC-4] packages/perps/SECURITY.md:384  "Opening-bounty release does not checkpoint carry." (the order path, when
//           the reward returns to the account that funded it: OrderRouter.sol:336-337)
//
// Save as packages/perps/test/PocS7_SelfTriggerCarry.t.sol and run from the repository root:
//   FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/PocS7_*' -vv
//
// Fixture: the repository's BasePerpTest (MockPyth and the USDG mock are the only mocks; both are outside the scope).
// Release risk parameters (DEPLOYMENT.md: carry 500 bps, trigger reward 0.20 USDG). No in-scope contract is modified.
// No vm.store, no vm.etch, no protocol address is impersonated. The test contract is the deployer; after setUp it
// makes no owner-only call. Every position is opened through commitOrder + executeOrder.
//
// Tests: 1 the owner triggers its own stop-loss (leading sequence); 2 control, another account triggers; 3 totals on
// both paths and each of the four close limits; 4 every other party; 5a / 5b who decides the reward recipient at the
// two creditBounty call sites; 6 elements of the impact rows; 7 control: a documented owner deposit gives the same end
// state (stated limit of the finding).
//
// Accounts:
//   ACCOUNT   the account that owns the position and its protection ("the owner" in the documents)
//   KEEPER    "another account": any caller, uses only permissionless entry points (trigger, executeOrder)
//   ACCOUNT_2 a second trader with an ordinary SHORT position of the same size, opened before ACCOUNT's LONG (the book
//             is balanced, so the close of the LONG has a positive price-impact charge and no rebate). It takes no part
//             in the sequence; test 5b uses one order of it for the order call site.

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console} from "forge-std/console.sol";

contract PocS7_SelfTriggerCarryTest is BasePerpTest {

    address internal constant ACCOUNT = address(0xA001);
    address internal constant ACCOUNT_2 = address(0xA002);
    address internal constant KEEPER = address(0x0E0E1);
    address internal constant JLP = address(0x1A01);
    address internal constant SLP = address(0x5A01);

    uint256 internal constant LOT = 1e20;
    uint256 internal constant MON = 1_710_151_200; // Mon 2024-03-11 10:00 UTC (live market)
    uint256 internal constant P0 = 1e8; // entry price 1.0000
    uint256 internal constant STOP = 1.01e8; // stop-loss of the LONG position: mark at or above 1.0100
    uint256 internal constant START_FUNDS = 2_000_000e6;
    uint256 internal constant LOTS = 10_000; // 1,000,000 USDG notional at 1.0000
    uint256 internal constant MARGIN = 100_000e6;
    uint256 internal constant ARMED_FOR = 30 days; // time between arming the protection and the stop being reached

    uint8 internal constant ST_TRIGGERED = uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered);
    uint8 internal constant ST_EXECUTED = uint8(PositionProtectionTypes.PositionProtectionStatus.Executed);
    uint8 internal constant ST_LATCHED = uint8(PositionProtectionTypes.PositionProtectionStatus.Latched);
    uint8 internal constant O_EXECUTED = uint8(OrderV3Types.LifecycleStatus.Executed);
    uint8 internal constant O_FAILED = uint8(OrderV3Types.LifecycleStatus.Failed);
    uint8 internal constant C_NOTIONAL = uint8(OrderV3Types.ConstraintKind.ExecutionNotional);
    uint8 internal constant C_GROSS = uint8(OrderV3Types.ConstraintKind.GrossAccountDebit);
    uint8 internal constant C_CHARGE = uint8(OrderV3Types.ConstraintKind.ActionCharge);
    uint8 internal constant C_FEES = uint8(OrderV3Types.ConstraintKind.ExplicitFees);

    IPositionProtectionBook internal book;
    IOrderLifecycleBook internal lifecycle;
    uint256 internal nonce;
    uint256 internal reward; // trigger reward (router.positionProtectionTriggerBountyAssets)

    /// @dev Worth of every party, USDG atoms (6 decimals).
    struct Parties {
        uint256 account; // settlement balance + trader claim of ACCOUNT
        uint256 keeper; // settlement balance of KEEPER
        uint256 treasury; // settlement balance of the protocol treasury
        uint256 poolCash; // USDG held by the HousePool (LP side)
        uint256 poolAssets; // HousePool.totalAssets()
    }

    /// @dev One run of: arm -> wait -> trigger (by ACCOUNT itself or by KEEPER) -> KEEPER executes the close.
    struct Run {
        uint256 carryDueAtTrigger; // pending carry of ACCOUNT right before the trigger transaction
        uint256 carryLeftAfterTrigger; // pending carry of ACCOUNT right after it
        int256 dAccountTrigger; // change of ACCOUNT's worth in the trigger transaction
        int256 dPoolCashTrigger; // change of the pool's USDG in the trigger transaction
        int256 dKeeperTrigger;
        uint8 closeStatus; // 2 = Executed, 3 = Failed
        uint8 protStatus;
        uint256 sizeLots;
        // receipt of the protected close
        uint8 failedConstraint;
        uint256 failedActual;
        uint256 failedLimit;
        uint256 rNotional;
        uint256 rGrossDebit;
        uint256 rChargeAssessed;
        uint256 rChargeCollected;
        uint256 rExecFee;
        uint256 rSpread;
        int256 rVpi;
        int256 rPnl;
        int256 rCarry;
        uint256 rCommitCarry; // receipt.commitment.carryCollectedAssets
        Parties start; // before the trigger transaction
        Parties end_; // after the close attempt
    }

    // ------------------------------------------------------------------------------------------------------
    // fixture
    // ------------------------------------------------------------------------------------------------------

    // release configuration (DEPLOYMENT.md: VPI 0.01, max skew 0.4, MM 10 bps, IM 20 bps, frozen-market margin 300 bps,
    // carry 500 bps, liquidation charge min 1 USDG / 10 bps, keeper share 50 %, protocol share 0)
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams(0.01e18, 0.4e18, 10, 20, 300, 500, 1e6, 10, 5000, 0);
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }

    function setUp() public override {
        super.setUp();
        book = router.positionProtectionBook();
        lifecycle = IOrderLifecycleBook(address(router.lifecycleBook()));
        reward = router.positionProtectionTriggerBountyAssets();
        _setSeniorCapacity(40_000_000e6, 8000); // release Senior cap 40M / 80 %
        _warpTo(MON);
        _pushMark(P0);
        uint256 a = _requestDeposit(JLP, juniorVault, 5_000_000e6);
        _warpTo(pool.lpEpochStart(a));
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(JLP, juniorVault, a);
        _warpTo(_now() + 60);
        _pushMark(P0);
        uint256 s = _requestDeposit(SLP, seniorVault, 5_000_000e6);
        _warpTo(pool.lpEpochStart(s));
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(SLP, seniorVault, s);
        _fundTrader(ACCOUNT, START_FUNDS);
        _fundTrader(ACCOUNT_2, START_FUNDS);
        _warpTo(MON + 3 hours);
        _pushMark(P0);
        _trade(ACCOUNT_2, CfdTypes.Side.SHORT, LOTS, MARGIN, P0);
        _trade(ACCOUNT, CfdTypes.Side.LONG, LOTS, MARGIN, P0);
    }

    function _now() internal view returns (uint256) { return vm.getBlockTimestamp(); }

    function _warpTo(uint256 t) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _liveData(uint256 p) internal view returns (bytes[] memory d) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            u[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(p)), 0, -8, vm.getBlockTimestamp(), engine.lastMarkTime())
            );
        }
        d = new bytes[](1);
        d[0] = abi.encode(u);
    }

    function _pushMark(uint256 p) internal {
        bytes[] memory d = _liveData(p);
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    function _requestDeposit(address lp, TrancheVault v, uint256 amt) internal returns (uint256 id) {
        settlementToken.mint(lp, amt);
        vm.startPrank(lp);
        settlementToken.approve(address(v), amt);
        id = v.requestDeposit(amt, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(address lp, TrancheVault v, uint256 id) internal returns (uint256 shares) {
        uint256 amt = v.claimableDepositRequest(id, lp);
        vm.prank(lp);
        shares = v.claimDeposit(id, amt, lp, lp);
    }

    function _openRequest(CfdTypes.Side side, uint256 lots, uint256 margin, uint256 target)
        internal
        returns (OrderV3Types.OrderRequest memory r)
    {
        nonce += 1;
        r.clientOrderId = keccak256(abi.encode("S7", nonce));
        r.side = side;
        r.sizeDelta = lots * LOT;
        r.marginDelta = margin;
        r.targetPrice = target;
        r.isClose = false;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(_now() + 60),
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

    /// @dev Note: the fixture's _mockPythUpdateData advances the clock by one second and one block.
    function _exec(uint64 id, uint256 p, address executor) internal returns (OrderV3Types.ExecutionResult memory r) {
        bytes[] memory upd = _mockPythUpdateData(p);
        vm.prank(executor);
        r = router.executeOrder(id, upd);
    }

    /// @dev An ordinary open of `who` (LONG: any price accepted with target 1), committed by `who`, executed by KEEPER.
    function _trade(address who, CfdTypes.Side side, uint256 lots, uint256 margin, uint256 p) internal {
        _warpTo(_now() + 1);
        _pushMark(p);
        OrderV3Types.OrderRequest memory r = _openRequest(side, lots, margin, side == CfdTypes.Side.LONG ? 1 : CAP_PRICE);
        vm.prank(who);
        uint64 id = router.commitOrder(r);
        OrderV3Types.ExecutionResult memory res = _exec(id, p, KEEPER);
        assertEq(uint8(res.status), O_EXECUTED, "setup trade executed");
    }

    function _sizeLots(address a) internal view returns (uint256) {
        (uint256 s,,,,,,) = engine.positions(a);
        return s / LOT;
    }

    /// @dev Carry the account owes right now: accrued since its last collection + recorded arrears.
    function _carryDue(address a) internal view returns (uint256) {
        return _expectedIndexedCarryAssets(a) + engine.unsettledCarryAssets(a);
    }

    function _parties() internal view returns (Parties memory p) {
        p.account = clearinghouse.balanceAssets(ACCOUNT) + engine.traderClaimBalanceAssets(ACCOUNT);
        p.keeper = clearinghouse.balanceAssets(KEEPER);
        p.treasury = clearinghouse.balanceAssets(engine.protocolTreasury());
        p.poolCash = settlementToken.balanceOf(address(pool));
        p.poolAssets = pool.totalAssets();
    }

    function _d(uint256 a, uint256 b) internal pure returns (int256) { return int256(a) - int256(b); }

    function _maxLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
        l = PositionProtectionTypes.CloseFinancialLimits(
            type(uint256).max, type(uint256).max, type(uint256).max, type(uint256).max
        );
    }

    /// @dev ACCOUNT arms a stop-loss at 1.0100 with the given close limits.
    ///      positional: (takeProfitTriggerPrice, stopLossTriggerPrice, closeLimits)
    function _arm(PositionProtectionTypes.CloseFinancialLimits memory l) internal returns (uint64 id) {
        _warpTo(_now() + 1);
        _pushMark(engine.lastMarkPrice());
        _warpTo(_now() + 1);
        vm.prank(ACCOUNT);
        id = book.createPositionProtection(PositionProtectionTypes.PositionProtectionParams(0, STOP, l));
    }

    /// @dev What the full close would be charged at the stop price in the book as it is NOW (the public evaluator's
    ///      view, taken on a throw-away branch: arm without limits, KEEPER triggers, read the assessment, roll back).
    function _quoteCloseAtStop() internal returns (OrderV3Types.ExecutionAssessment memory a) {
        uint256 snap = vm.snapshotState();
        uint64 id0 = _arm(_maxLimits());
        _warpTo(_now() + 1);
        bytes[] memory d = _liveData(STOP);
        vm.prank(KEEPER);
        uint64 id = book.triggerPositionProtection(id0, d);
        _warpTo(_now() + 2);
        a = policyEvaluator.assessCommittedOrder(address(engine), id, KEEPER, STOP);
        vm.revertToState(snap);
    }

    /// @dev arm (limits `l`) -> ARMED_FOR passes -> the mark reaches the stop -> `caller` triggers (one transaction)
    ///      -> two seconds later KEEPER executes the queued close at the stop price.
    function _run(address caller, PositionProtectionTypes.CloseFinancialLimits memory l) internal returns (Run memory R) {
        _startRecordingLogs();
        uint64 pid = _arm(l);
        _warpTo(_now() + ARMED_FOR);
        bytes[] memory d = _liveData(STOP);
        R.start = _parties();
        R.carryDueAtTrigger = _carryDue(ACCOUNT);
        vm.prank(caller);
        uint64 closeId = book.triggerPositionProtection(pid, d);
        Parties memory mid = _parties();
        R.carryLeftAfterTrigger = _carryDue(ACCOUNT);
        R.dAccountTrigger = _d(mid.account, R.start.account);
        R.dPoolCashTrigger = _d(mid.poolCash, R.start.poolCash);
        R.dKeeperTrigger = _d(mid.keeper, R.start.keeper);
        assertEq(uint8(book.getPositionProtection(pid).status), ST_TRIGGERED, "Triggered after the trigger");
        _warpTo(_now() + 1);
        OrderV3Types.ExecutionResult memory r = _exec(closeId, STOP, KEEPER);
        R.closeStatus = uint8(r.status);
        R.protStatus = uint8(book.getPositionProtection(pid).status);
        R.sizeLots = _sizeLots(ACCOUNT);
        R.end_ = _parties();
        OrderV3Types.OrderReceipt memory rc = _verifiedReceipt(lifecycle, closeId);
        R.failedConstraint = uint8(rc.failure.constraint);
        R.failedActual = rc.failure.actual;
        R.failedLimit = rc.failure.limit;
        R.rNotional = rc.economics.executionNotionalAssets;
        R.rGrossDebit = rc.economics.grossAccountDebitAssets;
        R.rChargeAssessed = rc.economics.actionChargeAssessedAssets;
        R.rChargeCollected = rc.economics.actionChargeCollectedAssets;
        R.rExecFee = rc.economics.executionFeeAssets;
        R.rSpread = rc.economics.frozenSpreadAssets;
        R.rVpi = rc.economics.vpiAssets;
        R.rPnl = rc.economics.realizedPnlAssets;
        R.rCarry = rc.economics.carryAssets;
        R.rCommitCarry = rc.commitment.carryCollectedAssets;
    }

    function _both(PositionProtectionTypes.CloseFinancialLimits memory l)
        internal
        returns (Run memory self_, Run memory keeper_)
    {
        uint256 snap = vm.snapshotState();
        self_ = _run(ACCOUNT, l);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        keeper_ = _run(KEEPER, l);
        vm.revertToState(snap);
    }

    /// @dev The owner's ceiling of the leading scenario: the close charge quoted when the protection is armed
    ///      + 25 % of the execution fee as headroom. Only maxActionChargeAssets is limited.
    function _ownerCeiling() internal returns (PositionProtectionTypes.CloseFinancialLimits memory l, uint256 quoted, uint256 fee) {
        OrderV3Types.ExecutionAssessment memory q = _quoteCloseAtStop();
        quoted = q.actionChargeAssessedAssets;
        fee = q.executionFeeAssets;
        l = _maxLimits();
        l.maxActionChargeAssets = quoted + fee / 4;
    }

    function _logRun(string memory tag, Run memory R) internal pure {
        console.log(string.concat("  [", tag, "] carry due before the trigger / carry left right after it"),
            R.carryDueAtTrigger, R.carryLeftAfterTrigger);
        console.log(string.concat("  [", tag, "] trigger transaction: change of account / pool cash / keeper"));
        console.logInt(R.dAccountTrigger);
        console.logInt(R.dPoolCashTrigger);
        console.logInt(R.dKeeperTrigger);
        console.log(string.concat("  [", tag, "] close status (2 = Executed, 3 = Failed) / protection status / position lots"),
            R.closeStatus, R.protStatus, R.sizeLots);
        console.log(string.concat("  [", tag, "] close receipt: action charge assessed / collected / gross account debit"),
            R.rChargeAssessed, R.rChargeCollected, R.rGrossDebit);
        console.log(string.concat("  [", tag, "] close receipt: execution fee / carry / commitment carry"), R.rExecFee,
            uint256(R.rCarry), R.rCommitCarry);
        console.log(string.concat("  [", tag, "] close receipt: price-impact charge (signed) / realized price PnL (signed)"));
        console.logInt(R.rVpi);
        console.logInt(R.rPnl);
        console.log(string.concat("  [", tag, "] account worth: before the trigger / after the close attempt"), R.start.account,
            R.end_.account);
        console.log(string.concat("  [", tag, "] close receipt: failed constraint (4 = ActionCharge) / actual / limit"),
            R.failedConstraint, R.failedActual, R.failedLimit);
    }

    // ------------------------------------------------------------------------------------------------------
    // 1. leading scenario: the owner triggers its own stop-loss
    // ------------------------------------------------------------------------------------------------------

    /// One transaction of the owner (the trigger). Its pending carry is collected inside that transaction; the close
    /// then executes although carry + close charge is above the owner's own maxActionChargeAssets.
    function test_1_OwnerTriggersOwnProtection_CarryCollectedOutsideCloseLimitsAndReceipt() public {
        (PositionProtectionTypes.CloseFinancialLimits memory l, uint256 quoted, uint256 fee) = _ownerCeiling();
        Run memory R = _run(ACCOUNT, l);
        console.log("  owner's ceiling: close charge quoted at arming / execution fee in it / maxActionChargeAssets", quoted,
            fee, l.maxActionChargeAssets);
        _logRun("owner triggers", R);

        // [DOC-1] says the trigger has no carry effects. Observed: the whole pending carry leaves the account in it.
        assertGt(R.carryDueAtTrigger, 0, "carry is pending before the trigger");
        assertEq(R.carryLeftAfterTrigger, 0, "the trigger transaction collected all of it");
        assertEq(R.dAccountTrigger, -int256(R.carryDueAtTrigger), "account: -carry (the reward returned to it)");
        assertEq(R.dPoolCashTrigger, int256(R.carryDueAtTrigger), "pool: +carry in the trigger transaction");
        // the close executes
        assertEq(R.closeStatus, O_EXECUTED, "close executed");
        assertEq(R.protStatus, ST_EXECUTED, "protection Executed");
        assertEq(R.sizeLots, 0, "account flat");
        // [DOC-2] says receipts and bounds include the direct collection once. Observed: the receipt has none of it.
        assertEq(R.rCommitCarry, 0, "receipt.commitment.carryCollectedAssets = 0");
        assertLt(uint256(R.rCarry), R.carryDueAtTrigger / 1000, "receipt carry = only the seconds after the trigger");
        assertLe(R.rChargeAssessed, l.maxActionChargeAssets, "the charge the close was checked with is within the ceiling");
        uint256 paid = R.carryDueAtTrigger + R.rChargeAssessed;
        assertGt(paid, l.maxActionChargeAssets, "carry + close charge is ABOVE the owner's ceiling");
        console.log("  carry collected in the trigger + close charge in the receipt = paid / above the ceiling by", paid,
            paid - l.maxActionChargeAssets);
    }

    // ------------------------------------------------------------------------------------------------------
    // 2. control: same state, same ceiling, another account triggers
    // ------------------------------------------------------------------------------------------------------

    function test_2_Control_AnotherAccountTriggers_SameCeilingRefusesTheClose() public {
        (PositionProtectionTypes.CloseFinancialLimits memory l,,) = _ownerCeiling();
        (Run memory S, Run memory K) = _both(l);
        _logRun("another account triggers", K);

        assertEq(K.carryDueAtTrigger, S.carryDueAtTrigger, "same pending carry on both paths");
        assertEq(K.carryLeftAfterTrigger, K.carryDueAtTrigger, "carry untouched by another account's trigger [DOC-1]");
        assertEq(K.dAccountTrigger, -int256(reward), "account pays only the trigger reward");
        assertEq(K.dKeeperTrigger, int256(reward), "the other account receives the reward");
        assertEq(K.dPoolCashTrigger, 0, "pool unchanged in the trigger transaction");
        assertEq(K.closeStatus, O_FAILED, "close refused");
        assertEq(K.failedConstraint, C_CHARGE, "on maxActionChargeAssets");
        assertEq(K.failedLimit, l.maxActionChargeAssets, "the owner's ceiling");
        assertGt(K.failedActual, l.maxActionChargeAssets, "assessed charge incl. carry is above it");
        assertEq(K.protStatus, ST_LATCHED, "protection Latched");
        assertEq(K.sizeLots, LOTS, "position still open");
        // same ceiling, same book, same price: only the caller of the trigger differs
        assertEq(S.closeStatus, O_EXECUTED, "owner-triggered close executed under the same ceiling");
        console.log("  same ceiling: close after another account's trigger / after the owner's trigger (2 = Executed, 3 = Failed)",
            K.closeStatus, S.closeStatus);
        console.log("  assessed charge refused on the other account's path / carry + charge paid on the owner's path",
            K.failedActual, S.carryDueAtTrigger + S.rChargeAssessed);
    }

    // ------------------------------------------------------------------------------------------------------
    // 3. totals on both paths and each of the four close limits
    // ------------------------------------------------------------------------------------------------------

    function test_3_Totals_BothPaths_AndEachOfTheFourLimits() public {
        (Run memory S, Run memory K) = _both(_maxLimits());
        console.log("TOTALS with unlimited ceilings (both closes execute)");
        _logRun("owner triggers", S);
        _logRun("another account triggers", K);
        assertEq(S.closeStatus, O_EXECUTED);
        assertEq(K.closeStatus, O_EXECUTED);
        // what the account bears besides the price loss, from its worth: -(change of worth) - price loss - reward paid away
        uint256 selfBorne = uint256(-_d(S.end_.account, S.start.account) + S.rPnl);
        uint256 otherBorne = uint256(-_d(K.end_.account, K.start.account) + K.rPnl) - reward;
        uint256 selfCarry = S.carryDueAtTrigger + uint256(S.rCarry);
        console.log("  charges borne by the account (carry + fee + price impact): owner path / other path", selfBorne, otherBorne);
        console.log("  carry paid: owner path (in trigger + in close) / other path (in close)", selfCarry, uint256(K.rCarry));
        console.log("  account end worth: owner path / other path", S.end_.account, K.end_.account);
        console.log("  trigger reward paid away: owner path / other path", uint256(0), reward);
        // the account bears the same charges on both paths; the only difference in its worth is the reward it kept
        assertEq(selfBorne, otherBorne, "same charges borne on both paths");
        assertEq(selfCarry, uint256(K.rCarry), "same carry paid on both paths");
        assertEq(_d(S.end_.account, K.end_.account), int256(reward), "worth differs by the reward only");
        // what the close was checked with and what its receipt shows
        assertEq(K.rChargeAssessed, otherBorne, "other path: the receipt's action charge = all charges borne");
        assertEq(selfBorne - S.rChargeAssessed, S.carryDueAtTrigger, "owner path: receipt's action charge is short by the carry");
        assertEq(K.rGrossDebit - S.rGrossDebit, S.carryDueAtTrigger, "owner path: receipt's gross debit is short by the carry");
        console.log("  owner path, receipt short by: action charge / gross account debit", selfBorne - S.rChargeAssessed,
            K.rGrossDebit - S.rGrossDebit);

        // each limit set ONE atom below what the close needs when another account triggers
        console.log("EACH LIMIT one atom below the other-account path's value: constraint that refuses there / owner-path close status");
        uint256[4] memory need = [K.rNotional, K.rGrossDebit, K.rChargeAssessed, K.rExecFee + K.rSpread];
        uint8[4] memory kind = [C_NOTIONAL, C_GROSS, C_CHARGE, C_FEES];
        bool[4] memory ownerPasses = [false, true, true, false];
        for (uint256 i; i < 4; ++i) {
            PositionProtectionTypes.CloseFinancialLimits memory l = _maxLimits();
            if (i == 0) l.maxExecutionNotionalAssets = need[i] - 1;
            if (i == 1) l.maxGrossAccountDebitAssets = need[i] - 1;
            if (i == 2) l.maxActionChargeAssets = need[i] - 1;
            if (i == 3) l.maxExplicitFeesAssets = need[i] - 1;
            (Run memory s, Run memory k) = _both(l);
            console.log("  limit index (0 notional, 1 gross debit, 2 action charge, 3 explicit fees) / limit", i, need[i] - 1);
            console.log("    other-account path: status / failed constraint; owner path: status", k.closeStatus, k.failedConstraint,
                s.closeStatus);
            assertEq(k.closeStatus, O_FAILED, "other-account path refused one atom below");
            assertEq(k.failedConstraint, kind[i], "by this limit");
            assertEq(s.closeStatus, ownerPasses[i] ? O_EXECUTED : O_FAILED, "owner path");
        }
    }

    // ------------------------------------------------------------------------------------------------------
    // 4. every other party, both paths
    // ------------------------------------------------------------------------------------------------------

    function test_4_OtherParties_PoolTreasuryKeeper_AndConservation() public {
        (Run memory S, Run memory K) = _both(_maxLimits());
        int256[5] memory s = [
            _d(S.end_.account, S.start.account),
            _d(S.end_.poolCash, S.start.poolCash),
            _d(S.end_.treasury, S.start.treasury),
            _d(S.end_.keeper, S.start.keeper),
            _d(S.end_.poolAssets, S.start.poolAssets)
        ];
        int256[5] memory k = [
            _d(K.end_.account, K.start.account),
            _d(K.end_.poolCash, K.start.poolCash),
            _d(K.end_.treasury, K.start.treasury),
            _d(K.end_.keeper, K.start.keeper),
            _d(K.end_.poolAssets, K.start.poolAssets)
        ];
        console.log("change from before the trigger to after the close (unlimited ceilings): owner path, then other-account path");
        console.log("  account");
        console.logInt(s[0]);
        console.logInt(k[0]);
        console.log("  pool cash (LP side)");
        console.logInt(s[1]);
        console.logInt(k[1]);
        console.log("  treasury");
        console.logInt(s[2]);
        console.logInt(k[2]);
        console.log("  keeper (the other account)");
        console.logInt(s[3]);
        console.logInt(k[3]);
        console.log("  pool totalAssets");
        console.logInt(s[4]);
        console.logInt(k[4]);
        // closed system on both paths: nothing is created or lost
        assertEq(s[0] + s[1] + s[2] + s[3], 0, "owner path: account + pool + treasury + keeper = 0");
        assertEq(k[0] + k[1] + k[2] + k[3], 0, "other path: account + pool + treasury + keeper = 0");
        // LP side and treasury end the same on both paths (to the atom of rounding); the reward is the only transfer
        assertApproxEqAbs(s[1], k[1], 2, "pool cash: same on both paths");
        assertEq(s[2], k[2], "treasury: same on both paths");
        assertEq(k[3] - s[3], int256(reward), "keeper: +reward only when it triggered");
        assertApproxEqAbs(s[0] - k[0], int256(reward), 2, "account: keeps the reward when it triggered itself");
    }

    // ------------------------------------------------------------------------------------------------------
    // 5. who decides the recipient at the two creditBounty call sites
    // ------------------------------------------------------------------------------------------------------

    /// Trigger site (OrderLiquidationBatchLogic.sol:617-619): the recipient is the caller of
    /// triggerPositionProtection (PositionProtectionBook.sol:376 appends msg.sender). Another account cannot make the
    /// protected account the recipient: when it triggers, the protected account's carry is not touched.
    function test_5a_TriggerSite_RecipientIsTheCaller_AnotherAccountCannotCollectTheOwnersCarry() public {
        uint64 pid = _arm(_maxLimits());
        _warpTo(_now() + ARMED_FOR);
        bytes[] memory d = _liveData(STOP);
        uint256 due = _carryDue(ACCOUNT);
        uint256 accBefore = clearinghouse.balanceAssets(ACCOUNT);
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        uint256 poolBefore = settlementToken.balanceOf(address(pool));
        vm.prank(KEEPER);
        book.triggerPositionProtection(pid, d);
        assertGt(due, 0);
        assertEq(_carryDue(ACCOUNT), due, "owner's pending carry unchanged");
        assertEq(accBefore - clearinghouse.balanceAssets(ACCOUNT), reward, "owner paid the reward only");
        assertEq(clearinghouse.balanceAssets(KEEPER) - keeperBefore, reward, "the caller received the reward");
        assertEq(settlementToken.balanceOf(address(pool)), poolBefore, "no carry reached the pool");
        console.log("  another account triggers: owner's carry due before / after", due, _carryDue(ACCOUNT));
    }

    /// Order site (OrderRouter.sol:336-342): the recipient is the caller of executeOrder. When that caller is the
    /// order's own account the reward is RELEASED WITHOUT a carry collection [DOC-4]; when it is another account the
    /// reward is credited to that caller and the order owner's carry is not touched either.
    /// The order used is an increase of ACCOUNT_2's SHORT whose target price cannot be met, so it ends Failed and
    /// only the reward moves.
    function test_5b_OrderSite_OwnRewardIsReleasedWithoutCarryCollection() public {
        _warpTo(_now() + ARMED_FOR);
        _pushMark(P0);
        _warpTo(_now() + 1);
        // SHORT increase with target 1: needs an execution price of at most 0.00000001, the oracle shows 1.0000
        OrderV3Types.OrderRequest memory r = _openRequest(CfdTypes.Side.SHORT, 100, 1_000e6, 1);
        vm.prank(ACCOUNT_2);
        uint64 id = router.commitOrder(r);
        uint256 orderReward = clearinghouse.getBountyReservation(IMarginClearinghouse.BountyKind.Order, id).amountAssets;
        assertGt(orderReward, 0, "the open order reserved a reward");
        _warpTo(_now() + 30);
        uint256 snap = vm.snapshotState();

        // (i) the order's own account executes it
        bytes[] memory upd = _mockPythUpdateData(P0);
        uint256 due = _carryDue(ACCOUNT_2);
        uint256 bal = clearinghouse.balanceAssets(ACCOUNT_2);
        uint256 poolBefore = settlementToken.balanceOf(address(pool));
        vm.prank(ACCOUNT_2);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(id, upd);
        assertEq(uint8(res.status), O_FAILED, "order failed (target not met)");
        assertGt(due, 0, "carry is pending");
        assertEq(_carryDue(ACCOUNT_2), due, "own execution: pending carry NOT collected");
        assertEq(clearinghouse.balanceAssets(ACCOUNT_2), bal, "own execution: balance unchanged (reward released back)");
        assertEq(settlementToken.balanceOf(address(pool)), poolBefore, "own execution: nothing reached the pool");
        console.log("  order site, own execution: carry due before / after / reward released", due, _carryDue(ACCOUNT_2), orderReward);
        vm.revertToState(snap);

        // (ii) another account executes it
        upd = _mockPythUpdateData(P0);
        due = _carryDue(ACCOUNT_2);
        bal = clearinghouse.balanceAssets(ACCOUNT_2);
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        vm.prank(KEEPER);
        res = router.executeOrder(id, upd);
        assertEq(uint8(res.status), O_FAILED, "order failed (target not met)");
        assertEq(_carryDue(ACCOUNT_2), due, "other executor: the order owner's carry NOT collected");
        assertEq(bal - clearinghouse.balanceAssets(ACCOUNT_2), orderReward, "other executor: owner pays the reward");
        assertEq(clearinghouse.balanceAssets(KEEPER) - keeperBefore, orderReward, "other executor: caller receives it");
        console.log("  order site, another executor: carry due before / after / reward paid", due, _carryDue(ACCOUNT_2), orderReward);
    }

    // ------------------------------------------------------------------------------------------------------
    // 6. elements of the impact rows
    // ------------------------------------------------------------------------------------------------------

    /// Low row "Contract fails to deliver promised returns, but doesn't lose value":
    ///   "fails to deliver promised ..."  = the owner's ceiling and the receipt do not cover the carry (asserted)
    ///   "doesn't lose value"             = closed system, every party except the reward ends the same (asserted)
    /// Rows above it need a party that ends with less than on the documented path, or funds that cannot be moved:
    ///   no such party exists here (asserted: pool, treasury and keeper end the same or better; the account pays the
    ///   same charges; its position closes; and another account cannot cause the collection).
    function test_6_ImpactRowElements() public {
        (PositionProtectionTypes.CloseFinancialLimits memory l,,) = _ownerCeiling();
        uint256 snap = vm.snapshotState();
        Run memory K = _run(KEEPER, l);
        vm.revertToState(snap);
        Run memory S = _run(ACCOUNT, l); // this branch stays live for the withdrawal below
        // promise not delivered: charge above the ceiling was taken, and it is not in the receipt
        assertGt(S.carryDueAtTrigger + S.rChargeAssessed, l.maxActionChargeAssets, "paid above the ceiling");
        assertEq(S.rCommitCarry, 0, "not in the receipt's commitment");
        assertLe(S.rChargeAssessed, l.maxActionChargeAssets, "not in the receipt's charge");
        // no value lost: closed system
        int256 sum = _d(S.end_.account, S.start.account) + _d(S.end_.poolCash, S.start.poolCash)
            + _d(S.end_.treasury, S.start.treasury) + _d(S.end_.keeper, S.start.keeper);
        assertEq(sum, 0, "account + pool + treasury + keeper = 0");
        // nobody else is worse off than on the other path (there the close is refused and the position stays open)
        assertGe(S.end_.poolCash, K.end_.poolCash, "pool cash not lower");
        assertGe(S.end_.treasury, K.end_.treasury, "treasury not lower");
        assertEq(K.end_.keeper - S.end_.keeper, reward, "keeper: only the reward it did not earn");
        // nothing is frozen: the account is flat and its balance is free to withdraw
        assertEq(S.sizeLots, 0);
        uint256 free = clearinghouse.getAccountAssetBuckets(ACCOUNT).freeSettlementAssets;
        assertEq(free, clearinghouse.balanceAssets(ACCOUNT), "whole balance is free settlement");
        vm.prank(ACCOUNT);
        clearinghouse.withdraw(ACCOUNT, free);
        assertEq(settlementToken.balanceOf(ACCOUNT), free, "withdrawn");
        console.log("  owner path under its own ceiling: paid above the ceiling by / account withdrew",
            S.carryDueAtTrigger + S.rChargeAssessed - l.maxActionChargeAssets, free);
    }

    // ------------------------------------------------------------------------------------------------------
    // 7. limit of the finding, stated as a control: a documented owner action gives the same end state
    // ------------------------------------------------------------------------------------------------------

    /// Another account triggers; before the close is executed the owner makes an ordinary deposit of 1 USDG. A deposit
    /// collects carry by its documented rule (CANONICAL_ENTRYPOINTS.md:31 "A deposit credits custody and checkpoints
    /// carry"). The close then executes under the same ceiling. So the owner can also move its carry out of the
    /// close's limits with a second, documented transaction; what differs in tests 1-3 is that the TRIGGER
    /// transaction itself does it, although [DOC-1] says the trigger has no carry effects.
    function test_7_Control_OwnerDepositAfterAnotherAccountsTrigger_SameEndState() public {
        (PositionProtectionTypes.CloseFinancialLimits memory l,,) = _ownerCeiling();
        _startRecordingLogs();
        uint64 pid = _arm(l);
        _warpTo(_now() + ARMED_FOR);
        bytes[] memory d = _liveData(STOP);
        uint256 due = _carryDue(ACCOUNT);
        vm.prank(KEEPER);
        uint64 closeId = book.triggerPositionProtection(pid, d);
        assertEq(_carryDue(ACCOUNT), due, "carry untouched by the other account's trigger");
        settlementToken.mint(ACCOUNT, 1e6);
        vm.startPrank(ACCOUNT);
        settlementToken.approve(address(clearinghouse), 1e6);
        clearinghouse.deposit(ACCOUNT, 1e6);
        vm.stopPrank();
        assertEq(_carryDue(ACCOUNT), 0, "the owner's deposit collected the carry (documented)");
        _warpTo(_now() + 1);
        OrderV3Types.ExecutionResult memory r = _exec(closeId, STOP, KEEPER);
        OrderV3Types.OrderReceipt memory rc = _verifiedReceipt(lifecycle, closeId);
        assertEq(uint8(r.status), O_EXECUTED, "close executed under the same ceiling");
        assertLe(rc.economics.actionChargeAssessedAssets, l.maxActionChargeAssets);
        assertGt(due + rc.economics.actionChargeAssessedAssets, l.maxActionChargeAssets, "carry + charge above the ceiling");
        console.log("  other account triggers, owner deposits 1 USDG, close: status / carry collected by the deposit / receipt charge",
            uint8(r.status), due, rc.economics.actionChargeAssessedAssets);
    }
}
```
