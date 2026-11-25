# \#98366 [SC-Low] Account for carry in delegated order limits and authenticated receipts

**Submitted on Nov 7th 2026 at 12:13:25 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98366
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Classification and recommendation boundary

This is one **Architectural Decentralization and Composability Insight** about
the composability contract between Plether's bounded V3 orders and external
smart-account policies. The Plether Scope page lists this category as the
Insight row `Architectural decentralization`.

Immunefi defines this category as an architectural enhancement that reduces
unsafe-composability risks. Its valid examples include explicit handling and
checks for external contract interactions. The recommendation here concerns an
ordinary public order-commit path used through a selector-limited smart account.
The trigger is `OrderRouter.commitOrder`, which any account may call for
itself. The functions that carry the omitted value are internal hops of that
call: `OrderLifecycleBook.recordCommitment` is limited to the Router, and
`MarginClearinghouse.lockReservedSettlement` to the Engine, the Router and its
protection Book. No owner, pauser, guardian or treasury action is involved. The
recommendation is not a validation check for a privileged or access-controlled
change operation.

I am not reporting a current Critical, High, Medium, or Low vulnerability,
selecting a vulnerability impact, or claiming theft, overcharge, insolvency,
freezing, griefing, or funds at risk. The carry moved by the demonstrated path
was already accrued and owed to the Pool. The informational security value is
that the protocol's declared per-order debit limits and authenticated receipts
do not account for that value movement. The Engine emits `CarryRealized` for
it, but an external account policy cannot use the declared limits to bound that
movement, and the authenticated receipt does not reconcile it.

The programme's Insight Reporting rule says runnable code is not required and
the PoC section must describe the conditions under which the Insight is
valuable. The Scope page's PoC section says: "These rules do not apply to
Insights, which follow the Insight Reporting rules on the Information tab."
This report uses that Insight route and includes no attachment or runnable PoC.

## Missing commitment-effect invariant

V3 requests contain inclusive financial ceilings, including
`maxActionChargeAssets` and `maxGrossAccountDebitAssets`. The source states
that zero is a real zero allowance. The programme's agent guide
(`packages/perps/WORKING_WITH_AI_AGENTS.md` lines 7-9) says the application
supplies wallet delegation, session keys, strategy policy and user approvals,
while the protocol supplies per-order limits and authenticated receipts.

For an ordinary open or same-side increase, however,
`OrderReservationAccounting._reserveExecutionBounty` records only:

```solidity
effects.bountyFromFreeAssets = bounty;
```

It then calls:

```solidity
clearinghouse.lockReservedSettlement(account, bounty);
```

`MarginClearinghouse.lockReservedSettlement` first calls
`_checkpointCarryBeforeMarginChange(account)`. The Engine computes accrued
carry, consumes available account backing, transfers collected settlement to
`HousePool`, and emits `CarryRealized`. That carry movement is not returned to
the Router's `effects` object.

`OrderLifecycleBook.recordCommitment` enforces the two relevant ceilings from
the supplied effects:

```solidity
uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;

if (gross > pending.bounds.maxGrossAccountDebitAssets) { ... }
if (effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets) { ... }
```

Because the ordinary-open effects still contain zero carry, an increase can be
accepted with:

```text
maxActionChargeAssets      = 0
maxExecutionBountyAssets   = exact opening bounty
maxGrossAccountDebitAssets = exact opening bounty
```

even though commitment also collects positive carry. The stored commitment
then authenticates `carryCollectedAssets == 0`.
`CfdOrderPolicyEvaluator.assessCommittedOrder` includes stored commitment
effects only for closes, so the missing open/increase effect is not recovered
when terminal evidence is produced.

The resulting invariant gap is:

```text
actual commitment debit
    = execution bounty + collected carry

authenticated and bounded commitment debit
    = execution bounty + reported carry
    = execution bounty
```

## Why the recommendation has concrete security value

A smart account can grant a session key only the bounded order-commit method,
without any withdrawal or arbitrary-call method. Its policy can inspect the
submitted calldata and require the gross ceiling to equal the exact bounty and
the action ceiling to equal zero. It cannot inspect a later internal side
effect that the same in-scope contracts omit from both limit enforcement and
authenticated evidence.

Local runs of the in-scope contracts (mock settlement token and mock oracle; a
test subclass of the Router that leaves the public V3 commit path unchanged)
showed that the bounded increase is accepted at commitment. The account debit
and the Pool inflow both equal the independently projected carry, and the
pending commitment reports zero carry. When the order was then left to expire,
its terminal receipt also reported zero commitment carry. A matched close with
zero action and gross ceilings reverts and rolls the carry checkpoint back. The
source treats carry as part of the bound domain: `maxGrossAccountDebitAssets`
is documented as covering settlement debit (`OrderV3Types.sol` lines 131-136),
and `OrderLifecycleBook.sol` line 98 states that pre-commitment custody equals
post-commitment custody plus collected carry. The difference is therefore
specific to the ordinary-open reservation branch.

The v2.0.0 event corpus of the Arbitrum Sepolia test deployment (Mock USDG;
the code is not deployed on mainnet) contains 31 successful ordinary
open/increase commitments, out of 135 open commitments in the corpus, with a
same-transaction nonzero `CarryRealized` event and stored
`commitment.carryCollectedAssets == 0`. The omitted amounts total 29.529305
Mock USDG, with a 13.824545 Mock USDG maximum observation. Those figures
establish that the accounting shape occurs on the listed test deployment; they
are not incremental loss or assets-at-risk estimates.

All causal source files are byte-identical between the v2.0.0 release commit
`1209ca8a5b94a4fb6323c7c4585846ed93db3932` and the frozen competition commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

## Exact recommendation

Enforce one invariant across commitment, assessment, and receipt construction:

> Every account-value movement caused while committing an order must appear in
> the commitment effects, be checked against the caller-authored financial
> ceilings, and be included exactly once in authenticated terminal evidence.

Concretely:

1. Make opening-bounty reservation return the carry realization result, or
   snapshot total/free settlement before and after
   `lockReservedSettlement`.
2. Populate `carryCollectedAssets`, `carryOutstandingAssets`,
   `settlementBeforeAssets`, `settlementAfterAssets`, and
   `freeSettlementAfterAssets` for ordinary opens/increases.
3. Apply the existing gross-debit and action-charge checks to those actual
   effects before an order can persist, so that exact `carry + bounty` and
   exact carry-action limits pass and one atom below either limit reverts the
   complete commitment.
4. Include the stored commitment effect exactly once during open/increase
   assessment and receipt construction.

This is one recommendation family enforcing one composability invariant, not a
batch of separate Insights.

## Public overlap and limitations

The closest public items are relevant but do not state this recommendation's
root:

- Cecuro LOW-53 concerns an open preview that omits the later bounty and then
  reverts; this path succeeds and omits collected carry from bounds/evidence.
- Cecuro LOW-34 concerns close assessment over-reporting pending carry; this
  path under-reports carry actually collected during an open/increase commit.
- Cecuro MED-9 concerns trader limits on internally generated protection
  closes, not ordinary-open bounty reservation.
- The October AI scan finding on stale cached marks underfunding open bounties
  cites the same `lockReservedSettlement` call but concerns how the bounty is
  priced, not carry.
- The October AI scan finding on pledge-funded close bounties cites
  `recordCommitment` but concerns close bounty funding, not carry collected by
  an open or increase.
- PR #71 documents delegated bounded authority, and PR #105 describes exact
  carry/bounty accounting and authenticated receipts, but neither identifies
  this violation or supplies a repair for it.

A review of the public items above, the programme's audit reports and the
public repository (history, issues and pull requests) found no item with the
same first faulty transition, ordinary-open trigger, delegated-policy
consequence, receipt consequence, and repair. The programme rule makes
duplicate Insight submissions invalid.


## Proof of Concept

## Proof of Concept

## Conditions under which this Insight is valuable

This architectural recommendation is valuable when all of the following are
true:

1. A wallet, smart account, session key, strategy, or approval layer relies on
   Plether's V3 request limits to bound the value movement authorized by one
   order commit.
2. The account has an existing position with positive collectible carry.
3. A delegate is permitted to submit an ordinary open or same-side increase but
   is not given a withdrawal or arbitrary-call capability.
4. The increase has a positive opening bounty, so ordinary bounty reservation
   calls `lockReservedSettlement` and checkpoints carry before locking the
   bounty.
5. The account policy permits a gross debit equal to the exact bounty and an
   action charge of zero. A positive `minPostSettlementBalanceAssets` is not an
   alternative bound for this case: a fresh request with a positive floor is
   rejected when the account already has a position
   (`OrderLiquidationBatchLogic.sol` lines 307-312).
6. The account or an integrator relies on the pending commitment or terminal
   receipt to reconcile the value movement caused by that authorized action.

The recommendation remains useful even though collected carry was already
owed: external authority is commonly bounded per action, and authenticated
evidence must describe the action's complete value movement for those bounds
to be enforceable and auditable.

Applied as recommended, an increase on a position with accrued carry is
accepted only if its gross and action ceilings cover that carry, as a close
already requires. Callers that set bounty-only ceilings would see such a
commitment revert and would size their ceilings to include accrued carry.

## Concrete source trace

The frozen path is deterministic:

1. `OrderLiquidationBatchLogic` validates the submitted execution-bounty and
   gross-debit limits, then registers the bounded request.
2. `OrderCommitHandler._createPendingOrder` calls
   `_reserveExecutionBounty` before recording the order.
3. The ordinary-open branch sets only `bountyFromFreeAssets` and calls
   `lockReservedSettlement`.
4. `MarginClearinghouse.lockReservedSettlement` checkpoints carry before
   changing the reserved bucket.
5. `CfdEngine.realizeCarryBeforeMarginChange` collects available accrued carry
   and transfers it to the Pool.
6. No collected-carry value returns to the Router, so
   `OrderLifecycleBook.recordCommitment` checks and stores zero carry.
7. `CfdOrderPolicyEvaluator` includes commitment effects only for closes, so an
   open/increase terminal receipt still reports zero commitment carry.

The decisive bounds are therefore evaluated against incomplete effects:

```text
reported action charge = 0
actual collected carry > 0

reported gross debit = opening bounty
actual gross debit = opening bounty + collected carry
```

## Target and control observations from local runs

The runs used the in-scope contracts at the frozen commit with a mock
settlement token and a mock oracle. The Router was a test subclass that leaves
the public V3 commit path unchanged. In the run with a delegating account, that
account was a small contract written for the run, not a deployed account. The
target began with an existing Long, one hour of positive accrued carry, and an
exact 0.20 USDG opening bounty. The runs measured:

```text
increase accepted with maxActionChargeAssets = 0
increase accepted with maxGrossAccountDebitAssets = opening bounty
account carry debit = independently projected accrued carry
Pool inflow = the same accrued carry
pending commitment carry = 0
terminal receipt commitment carry = 0 (the order was left to expire)
```

The delegating account exposed only a bounded commit method and no withdrawal
or arbitrary-call method. A matched close from the same starting condition and
with zero gross and action limits reverted with
`OrderLifecycleBook__CommitmentBoundExceeded`; the carry checkpoint rolled
back and no pending order remained. That control distinguishes the ordinary-
open effect omission from a design where carry is intentionally outside the
limits. The increase was not executed, so no execution-time limit was tested.

## Deployment applicability and non-claims

No file under `packages/perps/src` differs between the v2.0.0 release commit
`1209ca8a5b94a4fb6323c7c4585846ed93db3932` and the frozen competition commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The v2.0.0 event corpus of the
Arbitrum Sepolia test deployment (Mock USDG) independently shows 31 ordinary
open/increase transactions with nonzero carry realized during commitment while
the stored commitment reports zero carry.

This report does not claim that the 29.529305 Mock USDG aggregate is
incremental damage. It does not claim theft, overcharge, insolvency, freezing,
griefing, mainnet deployment, or current user funds at risk. It establishes the
conditions and concrete architectural value of making commitment bounds and
authenticated evidence complete for smart-account composability.

The programme's Insight route does not require runnable code. Its requirement
for this field is to describe the conditions under which the Insight is
valuable; those conditions and the supporting source/evidence chain are stated
above.

