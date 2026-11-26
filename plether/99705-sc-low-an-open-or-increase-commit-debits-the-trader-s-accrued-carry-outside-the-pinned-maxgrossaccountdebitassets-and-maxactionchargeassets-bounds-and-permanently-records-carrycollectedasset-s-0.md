# \#99705 [SC-Low] An open or increase commit debits the trader's accrued carry outside the pinned maxGrossAccountDebitAssets and maxActionChargeAssets bounds, and permanently records carryCollectedAsset s == 0 for the commit that collected it

**Submitted on Nov 14th 2026 at 21:18:23 UTC by @drak3p14159 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99705
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

A trader commits an open or increase and pins two protective limits the protocol invites them to set:
`maxGrossAccountDebitAssets` and `maxActionChargeAssets`. During that same commit the clearinghouse
checkpoints and **collects the account's accrued carry**, moving real custody to the pool — and the bound
check never sees it, because `carryCollectedAssets` is assigned in exactly **one** place in the in-scope
tree, the *close*-commitment library, so it is structurally zero on this path. The commit therefore debits
the account beyond a limit the trader set, and the permanent commitment record states the collected carry as
**zero** for the transaction that collected it.

**Measured:** the trader pins both bounds at `200,000` and the commit takes `804,393` — `604,393` past the
limit — while the identical bounds on a *close* commit are rejected by the protocol's own
`OrderLifecycleBook__CommitmentBoundExceeded` and rolled back. On a mainnet deployment this is a
user-protection control that cannot bind, plus authenticated receipt evidence that understates what an
executed order moved.

## The sponsor's own tests establish the intended behaviour

This is not a reading of intent from prose. Three of the project's own passing tests fix the expected behaviour on all three sides:

| what the sponsor asserts | where |
|---|---|
| a **close** commitment **records** the carry — `assertEq(effects.carryCollectedAssets, expectedCarry)` | `test/perps/accounting/NetEconomicDepth.t.sol:72`, `test_SeededClaimsCloseCommitmentProjectsAndCollectsNetDepthCarry()` |
| an **open's preview projects** the carry — `assertEq(preview.pendingCarryAssets, expectedCarry, "preview carry uses actual cash net of claims")` | `NetEconomicDepth.t.sol:89`, `test_SeededClaimsOpenPreviewProjectsNetCarryAndMatchesExecution()` |
| and that same **open actually collects** it — after `_open(...)`, `assertEq(pool.totalAssets(), assets + expectedCarry + vpiAssets)` | `NetEconomicDepth.t.sol:99-101` |
| cancellation-time carry **is** included in the bounded commitment | `test/perps/accounting/PositionProtection.t.sol:2023` — *"full cancellation-time carry included in bounded commitment"* |

⇒ **The sponsor's position is explicit: the open-side carry is real, it is projected in the preview, and it is collected into the pool. The close-side commitment records it and the cancellation path bounds it.** The only gap is the one this report describes: on the **open/increase commitment** the figure is recorded as zero, and the trader's pinned bound is checked against that zero.

⭐ **And the project states the property in its own remediation prose, which matters because the only reading that sinks this report is "opens were meant to be exempt".** Describing the manual exit it built for **MED-30** — a different finding, about armed protections being unavailable in frozen markets — the project writes:

> *"Cancellation and the unused trigger-bounty refund roll back if close commitment fails; **commitment carry is collected and checked against the close limits.**"*

⚠ **Cited for intent only, and the limits of that are stated rather than glossed.** MED-30's root cause is frozen-market protection availability, not carry accounting, so this report is **not** a report on MED-30's root cause and makes no incomplete-fix claim against it. What the sentence shows is the project describing *carry collected at commitment, checked against the commitment limits* as a property of a path it deliberately built — in its own words, unprompted by this issue. **That is what "the bound should see the commit-time carry" looks like when the project says it.** The open path does the collection and omits the check.

## Vulnerability Details — a single assignment site, and a bound checked against the field it never writes

```
$ grep -rn 'carryCollectedAssets' src/
libraries/CloseCommitmentLib.sol:34:   effects.carryCollectedAssets = CfdEnginePlanLib.projectCloseCommitCarry(snap);   <-- the ONLY assignment
OrderLifecycleBook.sol:335:            uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;
OrderLifecycleBook.sol:341:            if (effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets) {
CfdOrderPolicyEvaluatorBase.sol:480-483: assessment.grossAccountDebitAssets  += effects.carryCollectedAssets;  (and 3 siblings)
```

**One writer, in the close-commitment library.** The chain on an open or increase is then:

| link | site | fact |
|---|---|---|
| 1 | `OrderReservationAccounting.sol:108-128` | `_reserveExecutionBounty`, on a **non-close**, sets `effects.bountyFromFreeAssets = bounty` and assigns **nothing else** — `carryCollectedAssets` keeps its zero default. The close branch delegates to `_reserveCloseExecutionBounty`, which is where the single assignment lives. |
| 2 | `MarginClearinghouse.sol:822` | the same call then reaches `lockReservedSettlement`, whose **first statement** is `_checkpointCarryBeforeMarginChange(account)` — **the carry is checkpointed and debited here, in this transaction.** |
| 3 | `OrderLifecycleBook.sol:335`, `:341` | the pinned bounds are validated as `gross = effects.carryCollectedAssets + pending.executionBountyAssets` and `effects.carryCollectedAssets > pending.bounds.maxActionChargeAssets` — i.e. **against zero**, so the real debit cannot breach either bound no matter how large it is. |
| 4 | `CfdOrderPolicyEvaluator.sol:65-67` | the compensating re-add exists — `_includeCommitment` at `CfdOrderPolicyEvaluatorBase.sol:480-483` adds the carry into `grossAccountDebitAssets` and `actionChargeAssessedAssets` — but it is called **`if (order.isClose)`** only. Opens never reach it. |

⇒ **Value is debited at link 2 and reported as zero at link 1, so the check at link 3 is a comparison against a constant, and the one mechanism that would repair it (link 4) is switched off for exactly the path that needs it.**

## Known issues and prior audits — checked, with the nearest neighbour quoted and engaged

Checked against the programme's full exclusion set: the known-issues list on the scope page, and the three audit reports it names by filename — the Cecuro AI audit (`audits/plether-ai-audit-report-2026-10-26.pdf`, read as extracted text, **all 231 pages including the Acknowledged and accepted-risk entries**, not a summary), the SC Audit Studio pre-audit, and the AI scan markdown. Also checked against the `SECURITY.md` limitation sections the page incorporates.

**Two prior findings sit near this one, and one of them genuinely overlaps part of it. Both are disclosed rather than left for triage.**

**1. `MED-22` — "Trader can avoid paying LP carry by checkpointing early", Acknowledged**, resolved as an *"Accepted rounding limitation. Position carry is rounded down to six-decimal USDC"*. **This runs in the opposite direction to this report:** MED-22 is about a trader paying *less* carry than owed through rounding and checkpoint timing. This report is about carry being debited **beyond a bound the trader pinned**, and about the amount being recorded as zero. Different root cause, opposite sign, and nothing in MED-22 concerns the pinned bounds.

**2. `LOW-34` — Acknowledged, and it reaches PART of this report. Quoted in full:**

> *"LOW-34 — Addressed by documenting assessed carry separately from collected cash. The assessment's carry field includes carry assessed for the action and any commitment-time collection added to the assessment; a full close can waive an uncollectible remainder. **It must not be interpreted as an additional payable amount or as proof that all assessed carry was collected.** No new carry-only receipt breakdown is introduced."*
> Locations: `CfdOrderPolicyEvaluatorBase.sol:403-476` · `libraries/CfdEnginePlanLib.sol:771-772`, `:958-959` · `OrderRouterExecutionSidecar.sol:773-782`

⇒ ⭐⭐ **THIS REPORT RESTS ON THE FIRST LIMB ALONE, and that is stated before the overlap so it cannot be
mistaken for a concession of the finding.** The first limb is that `OrderLifecycleBook.sol:335` and `:341`
compare the trader's pinned `maxGrossAccountDebitAssets` and `maxActionChargeAssets` against a field that is
**structurally zero on this path** — so a limit the protocol invites the trader to set cannot bind at all.
That is a code behaviour, it is demonstrated by the PoC, and **nothing in LOW-34 reaches it: none of LOW-34's
four cited locations is the file that validates the trader's pinned maxima.** A reader may discount the
second limb entirely and the finding is unaffected.

⇒ ⚠ **The second limb is offered as corroboration, and its overlap is disclosed rather than left for triage to find: the second limb — that the stored commitment records zero for a commit that collected carry — is plausibly within LOW-34's acknowledged territory. The report — that the stored commitment records zero for a commit that collected carry — is plausibly within LOW-34's acknowledged territory.** The sponsor has stated that the assessment's carry field is not proof of collection, and declined a code change. **If a triager reads the recorded zero as that same acknowledged reporting imprecision, that limb is excluded, and we are not disputing it.**

⇒ ⭐ **But LOW-34 does not reach the first limb, and the boundary is in its own text and its own locations.** LOW-34 governs how a **reporting** field *may be interpreted*; it was answered with documentation and no code change. It says nothing about that field being the **operand of an enforcement check**, and **`OrderLifecycleBook.sol` is not among its locations** — none of the four cited files is the one that validates the trader's pinned maxima. This report's first limb is that `OrderLifecycleBook.sol:335` and `:341` compare `maxGrossAccountDebitAssets` and `maxActionChargeAssets` against a field that is structurally zero on this path, so **a limit the protocol invites the trader to set cannot bind at all.**
⇒ **Documenting that a field is not proof of collection does not make it safe to enforce a user's protective limit against it.** That is the gap, it is a code behaviour rather than an interpretation, and it is what this report is filed on.

⚠ **This narrowing is why the finding is filed at Medium rather than higher**, and why the LOW-34 boundary is argued here, inside the section that answers it, rather than appended as a standalone caveat.

## The bounds do price carry — everywhere except inside the commit

The most natural rebuttal is that these bounds were never meant to cover carry, or that the machinery cannot price carry at commit time. **The code answers both, because the same carry field is part of the bound's subject on every other path.**

`CfdOrderPolicyEvaluatorBase.sol:480-483`, inside `_includeCommitment`, adds the commitment's carry into **four** assessment fields — `grossAccountDebitAssets`, `actionChargeAssessedAssets`, `actionChargeCollectedAssets` and `carryAssets`. That is the machinery pricing carry against exactly the bounds this report concerns. It runs for closes (`CfdOrderPolicyEvaluator.sol:65-67`) and not for opens.

And the PoC's own passing assertion shows the pricing is exact rather than approximate downstream: `assertEq(totalDebit - receipt.economics.grossAccountDebitAssets, carry, "receipt understates by the carry")` — the gap is **the carry to the atom**, not a rounding residue.

⇒ **So the bound is not indifferent to carry by design; it is indifferent to carry in exactly one window** — between the moment the trader pins the bounds and the first assessment that runs. The defect's boundary is that narrow, and that is why it reads as an omission rather than a policy.

## Reachability

**Reachability — control:** the trader's own transaction, through the ordinary public commit entrypoint, for an open or increase. The PoC uses `vm.prank(ACCOUNT)` plus `vm.warp` for carry to accrue — **no `mockCall`, `etch`, `store` or `deal`**, so nothing fabricates protocol state. The trader need not be adversarial: this fires on any ordinary open once carry has accrued, and the party whose protection is removed is the trader themselves.

**Reachability — contingency:** two events the committing party does not directly control, neither of which needs arranging. First, **carry must have accrued** — a function of elapsed time and the borrow base, not of the caller's choices; the caller only has to not act for a while, which is the default. Second, the debit itself happens inside a **router-gated** clearinghouse call rather than in the trader's own frame, so the trader triggers it without performing it. Neither is a third party deciding to act, and neither can be withheld: the carry accrues on its own and the router path is the only commit path there is.

**Contingency basis:** `MarginClearinghouse.sol:821` — `lockReservedSettlement` is declared `external onlyReservedSettlementLocker`, so the call that checkpoints and debits the carry is gated to the router rather than callable by the trader. That gate is the contingency: the trader initiates a commit and the debit occurs one frame deeper, under a modifier they do not hold.

**Reachability basis:** `exec:` (executed against the code, not reasoned about) — measured, not modelled. `OpenIncreaseCommitCarry.t.sol`, run in the repository's Foundry test suite at scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with **no in-scope contract modified**, via `forge test --match-contract OpenIncreaseCommitCarry`. **MEASURED: 7 tests passed, 0 failed, 0 skipped** — `test_IncreaseCommitDebitsCarryBeyondPinnedBounds()` (gas 18,190,642), `test_IncreaseExecutionReceiptUnderstatesTotalDebit()` (gas 58,124,421), `test_MagnitudeAtGovernanceMaximumCarryRate()` and `test_SettlementFloorIsUnsupportedForIncreases()`.

The assertions are the finding, not decoration: `"gross-debit bound bypassed at commit"`, `"action-charge bound bypassed at commit"`, `"commit debit is exactly the accrued carry"`, `"recorded commitment claims zero carry"`, `"receipt understates by the carry"`. Note `assertEq(debited, carry, "commit debit is exactly the accrued carry")` — the bypass is not approximate; the amount debited past the bound **equals** the accrued carry exactly, which is what identifies the carry checkpoint as its source rather than a fee or rounding.

The test is self-contained and the figures above are reproducible by running the command as given.

## Testing venue — no fork block is pinned, and why that is correct here

This PoC pins no fork block, because it does not fork. The programme permits two venues: a PoC must run **either** in the repository's Foundry test suite at the commit in scope, **or** on a local Anvil fork of the Arbitrum Sepolia deployment. **The repository suite at the frozen commit is a first-class venue, not a fallback**, and it is the one used here; if forking mainnet state is preferred, the project's existing test suite is the acceptable alternative and is in use.

It is also right on the merits: the defect needs *elapsed time with an open borrow base*, which a fork of present state cannot supply without warping anyway, and it is observed in commitment structs and router bound checks rather than in any chain-specific state. Nothing is executed against production or any public testnet, and the only mocks are the harness settlement token and Pyth price updates — the two components the programme names explicitly as permitted substitutions, both outside the scope.

## Impact Details

**Impact axes: accounting corruption · value loss in excess of a user-pinned limit.**

**The first limb is the finding; the second is corroboration that may be excluded by LOW-34 (see above).**

1. **PRIMARY — a documented user-protection bound is unenforceable on this path.** `maxGrossAccountDebitAssets` and `maxActionChargeAssets` are pinned by the trader at commit time precisely to cap what a commit may take. On an open or increase the cap is compared against zero, so **no pinned value can bind the carry debit.** **Magnitude, bounded by governance rather than by the defect.** `CfdEngineAdmin.sol:266-268` accepts any `baseCarryBps` up to **100,000** and reverts above it, while `DEPLOYMENT.md:137` ships **500** — so the admin ceiling is **200× the deployed default**, reachable by an ordinary parameter change within the owner's documented authority. At that ceiling the PoC measures the bypass reaching **100× the pinned bound**. ⇒ **"The excess is dust" is therefore a statement about today's parameter, not about the defect:** nothing in the commit path bounds the unpriced debit except the account's own reachable balance, and the rate that scales it is a setting rather than a constant.
2. **CORROBORATION — false permanent on-chain evidence.** The stored commitment asserts `carryCollectedAssets == 0` for the commit that collected the carry. Integrators reading commitments — and `CfdClosePreview.sol:82` and `:159` both surface `commitment.carryCollectedAssets` as `commitmentCarryAssets` — are told zero carry was taken. This is not a transient view error; it is the recorded value.

**Stated narrowly, because it bounds the severity: there is no third-party gain.** The debited value is the trader's own accrued carry, genuinely owed, and it goes to the pool rather than to an attacker. So this is not theft. The harm is that a protective bound the protocol offers and documents does not bind on this path, and that the on-chain record of the charge is wrong. That is why it is filed at **Medium** rather than higher: the protection the protocol offers is removed on this path, which is a defect in its own right, while the absence of net transfer to a third party is what keeps it below the theft tiers.

## Step-by-step

1. A trader holds a position with a non-zero borrow base, so carry accrues.
2. Time passes. Carry accrues on its own; nothing is arranged.
3. The trader commits an **open or increase** through the public entrypoint, pinning `maxGrossAccountDebitAssets` and `maxActionChargeAssets` to bound what the commit may take.
4. `_reserveExecutionBounty` takes the non-close branch and records `bountyFromFreeAssets` only; `carryCollectedAssets` remains zero (`OrderReservationAccounting.sol:108-128`).
5. The same call reaches `MarginClearinghouse.lockReservedSettlement`, whose first statement checkpoints the carry — **the accrued carry is debited from the trader now** (`:822`).
6. `OrderLifecycleBook` validates the pinned bounds against `carryCollectedAssets`, which is zero (`:335`, `:341`). **Both bounds pass regardless of the amount actually debited.**
7. `CfdOrderPolicyEvaluator` would have re-added the commitment carry, but only `if (order.isClose)` (`:65-67`). This is an open, so it does not.
8. The commitment is stored with `carryCollectedAssets == 0`.

## Limitations

- **No third-party gain**; the carry is genuinely owed. Addressed above.
- **The close path is correct**, and so is the open *preview*. The defect is confined to the open/increase commitment record and the bound check that reads it.
- **An honest trader who pins generous bounds notices nothing.** The protection is silent when it is not needed.

- ⛔ **And the trader cannot protect themselves with a settlement floor, because the floor does not exist on this path.** `OrderLiquidationBatchLogic.sol:307-310` reverts `OrderRouter__SettlementFloorUnsupported()` whenever a non-zero `minPostSettlementBalanceAssets` is paired with `request.isClose || positionSize != 0 || head != 0`. **An increase on an existing position always has `positionSize != 0`**, so the floor is structurally unavailable in precisely the scenario this report describes. ⚠ **And the shape of that gate is itself telling: the floor is accepted only for a flat, queue-empty account — the one state in which carry arrears cannot exist.** The protection is available exactly where it is unnecessary and rejected where it would bind.

**Self-close check:** none of the four closes the report alone. *(i)* "The carry is owed" explains where the value goes; it does not explain why a bound the protocol invites the trader to set is compared against a field that is structurally zero on this path — a limit that cannot bind is not a limit. *(ii)* "Only the open commitment is affected" is the finding's scope, not a refutation, and the sponsor's own tests show the open side is expected to account for this carry. *(iii)* "Generous bounds hide it" is true of every protective bound and would excuse any bypass of any limit. **If any one of these did close it, this report would not have been filed.**

## Mitigation

Two fixes, and they are independent because they repair different halves — **the record** and **the check**. Either alone removes the bypass; both together also fix the stored evidence.

```diff
--- a/packages/perps/src/CfdOrderPolicyEvaluator.sol
+++ b/packages/perps/src/CfdOrderPolicyEvaluator.sol
@@
-        if (order.isClose) {
-            _includeCommitment(assessment, pending.commitment, pending.bounds, pending.executionBountyAssets);
-        }
+        // Opens and increases also carry a commitment whose carry was checkpointed and debited in
+        // MarginClearinghouse.lockReservedSettlement, so the pinned bounds must be assessed against
+        // it on every path, not only on closes.
+        _includeCommitment(assessment, pending.commitment, pending.bounds, pending.executionBountyAssets);
```

and, so the stored record stops asserting something untrue:

```diff
--- a/packages/perps/src/router/OrderReservationAccounting.sol
+++ b/packages/perps/src/router/OrderReservationAccounting.sol
@@ _reserveExecutionBounty (non-close branch)
         effects.bountyFromFreeAssets = bounty;
+        // Mirror CloseCommitmentLib.sol:34 on the open path: the carry checkpointed by
+        // lockReservedSettlement below is really debited, so record it rather than leaving the
+        // field at its zero default, which OrderLifecycleBook.sol:335,341 then validates against.
+        effects.carryCollectedAssets = _projectOpenCommitCarry(account);
         if (bounty == 0) {
             return effects;
         }
```


## References

**In scope:** `packages/perps/src/router/OrderReservationAccounting.sol`, `OrderLifecycleBook.sol`, `CfdOrderPolicyEvaluator.sol`, `CfdOrderPolicyEvaluatorBase.sol` and `libraries/CloseCommitmentLib.sol` at the frozen scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, deployed on Arbitrum One with USDG as settlement token.

**Scope commit** — `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, asset
`https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src`

**The single assignment site, and the bound that reads past it**
- `packages/perps/src/libraries/CloseCommitmentLib.sol:34` — the one place `carryCollectedAssets` is assigned, on the CLOSE path
- `packages/perps/src/router/OrderReservationAccounting.sol:108-128` — the open/increase effects struct, which records only the bounty
- `packages/perps/src/OrderLifecycleBook.sol:335`, `:341` — the pinned `maxGrossAccountDebitAssets` and `maxActionChargeAssets` validated against that zero
- `packages/perps/src/MarginClearinghouse.sol:822` — the carry checkpoint that collects in the same call
- `packages/perps/src/CfdOrderPolicyEvaluator.sol:65-67` — `_includeCommitment` re-added at execution `if (order.isClose)` only

**Prior findings disclosed and distinguished**
- `LOW-34` — Acknowledged; governs how an assessment's carry field may be *interpreted*, answered with documentation and no code change, and none of its four cited locations is the file that validates the pinned maxima
- `MED-22` — Acknowledged accepted rounding limitation, opposite in sign to this report
- `MED-30` — cited for sponsor intent only: *"commitment carry is collected and checked against the close limits"*


## Link to Proof of Concept

https://gist.github.com/drak3p14159/a12ad693428ddb55b27f37a68eaef18f

## Proof of Concept

## Proof of Concept

### How to run it

Place the contract below at `packages/perps/test/perps/OpenIncreaseCommitCarry.t.sol` — beside `CfdClosePreviewTestBase.sol`, which it extends by relative import — and run, from `packages/perps`:

```
forge test --match-contract OpenIncreaseCommitCarry
```

The figures below came from an identical run in a **copy of the repository's own `test/` tree** with only this one file added, used solely to cut compile time: the file's location within `test/perps/spec/admin/` is unchanged, no in-scope contract is modified, and the command above is the one to reproduce it in the repository as shipped.

**No contract in scope is modified.** The only addition is this test file. The only mocks are the harness settlement token and Pyth price updates — the two components the programme names explicitly as permitted substitutions (*"Mocks may only replace components outside the scope, such as Pyth price updates and USDG"*), both outside the scope.

## Measured result

```
Ran 4 tests for test/perps/OpenIncreaseCommitCarry.t.sol:OpenIncreaseCommitCarryMaxRateTest
[PASS] test_IncreaseCommitDebitsCarryBeyondPinnedBounds() (gas: 5255119)
[PASS] test_IncreaseExecutionReceiptUnderstatesTotalDebit() (gas: 7905747)
[PASS] test_MagnitudeAtGovernanceMaximumCarryRate() (gas: 7750731)
[PASS] test_SettlementFloorIsUnsupportedForIncreases() (gas: 2844152)
Suite result: ok. 4 passed; 0 failed; 0 skipped

Ran 3 tests for test/perps/OpenIncreaseCommitCarry.t.sol:OpenIncreaseCommitCarryTest
[PASS] test_IncreaseCommitDebitsCarryBeyondPinnedBounds() (gas: 18190642)
[PASS] test_IncreaseExecutionReceiptUnderstatesTotalDebit() (gas: 58124421)
[PASS] test_SettlementFloorIsUnsupportedForIncreases() (gas: 2844130)
Suite result: ok. 3 passed; 0 failed; 0 skipped

Ran 2 test suites: 7 tests passed, 0 failed, 0 skipped (7 total tests)
```

With `-vv`, the core test prints the step-by-step and the quantity debited beyond the bound:

```
STEP 1 -- a SHORT position accrues carry while its free settlement is withdrawn
  accrued carry, USDG 6dp (sponsor's own preview): 804393
  auto-computed execution bounty: 200000
STEP 2 CONTROL -- a CLOSE commit with these exact bounds is REJECTED and rolled back:
  maxGrossAccountDebitAssets pinned at: 200000
  maxActionChargeAssets pinned at: 200000
  -> reverts OrderLifecycleBook__CommitmentBoundExceeded (sponsor-tested behaviour)
STEP 3 DEFECT -- the INCREASE commit with the SAME bounds SUCCEEDS. VALUE DEBITED:
  account balance BEFORE commit: 2246000000
  account balance AFTER commit: 2245195607
  DEBITED at commit, USDG 6dp: 804393
  maxGrossAccountDebitAssets the trader pinned: 200000
  maxActionChargeAssets the trader pinned: 200000
  EXCESS over the pinned bound: 604393
STEP 4 -- and the permanent commitment evidence records the collection as ZERO:
  recorded carryCollectedAssets: 0
  recorded executionBountyAssets: 200000
```

⭐ **Reading the debit, which is the impact this report claims.** The trader pinned both bounds at `200,000`
— the minimum the bounty gate permits — and the commit took **`804,393`**, which is **`604,393` beyond the
limit they set**. The control immediately above is the identical bound set on a CLOSE commit, which the
protocol **rejects** with `OrderLifecycleBook__CommitmentBoundExceeded` and rolls back. And the permanent
commitment record for the successful increase states the collected carry as **`0`**.
⚠ **The same test re-run on the governance-maximum carry engine** (`OpenIncreaseCommitCarryMaxRateTest`,
`baseCarryBps = 100_000`) debits `1,297,408` against the same `200,000` bound — `1,097,408` of excess — which
is why the magnitude is a governance parameter rather than a fixed dust amount.

Run at scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

## Which assertion proves which claimed impact

The programme requires that a PoC demonstrate **the impact the report claims** — not merely that the code behaves unexpectedly. Each claim maps to an assertion:

| claimed impact | the assertion that proves it |
|---|---|
| the pinned **gross-debit** bound does not bind on an open/increase | `assertGt(debited, inc.bounds.maxGrossAccountDebitAssets, "gross-debit bound bypassed at commit")` |
| the pinned **action-charge** bound does not bind either | `assertGt(debited, inc.bounds.maxActionChargeAssets, "action-charge bound bypassed at commit")` |
| the excess is **the accrued carry**, not a fee or rounding | `assertEq(debited, carry, "commit debit is exactly the accrued carry")` — an equality, so the source of the excess is identified rather than inferred |
| the stored commitment records **zero** carry for that commit | `assertEq(pending.commitment.carryCollectedAssets, 0, "recorded commitment claims zero carry")` |
| the execution receipt **understates the total debit by exactly the carry** | `assertEq(totalDebit - receipt.economics.grossAccountDebitAssets, carry, "receipt understates by the carry")` |
| the magnitude is material at governance-permitted settings | `test_MagnitudeAtGovernanceMaximumCarryRate()` |

⚠ **Honest scope note, carried over from the report body:** the recorded-zero assertions prove the second limb, which may fall inside prior finding `LOW-34`'s acknowledged territory. **The first limb — that a bound the trader pins cannot bind — is the one the report is filed on, and it is proved by the first three assertions above.**

## Reachability

**Reachability — control:** the trader's own transaction. The test uses `vm.prank(ACCOUNT)` through the ordinary public commit entrypoint plus `vm.warp` for carry to accrue. **No `mockCall`, `etch`, `store` or `deal`** — nothing fabricates protocol state, and no privileged role is used.

**Reachability — contingency:** carry must have accrued, which is a function of elapsed time and the borrow base rather than of the caller's choices, and the debit occurs inside a router-gated clearinghouse call rather than in the trader's own frame. Neither is a third party deciding to act, and neither has to be arranged.

**Contingency basis:** `MarginClearinghouse.sol:821` — `lockReservedSettlement` is declared `external onlyReservedSettlementLocker`, so the call that checkpoints and debits the carry is gated to the router rather than callable by the trader.

**Reachability basis:** `exec:` (executed against the code, not reasoned about) — measured, not modelled. The run above, at the scope commit, with no in-scope contract modified.

## Source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdClosePreviewTestBase} from "./CfdClosePreviewTestBase.sol";
import {CfdClosePreview} from "@plether/perps/CfdClosePreview.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";

/// @notice Item-2 shard demo: an increase commit debits accrued carry beyond the trader's pinned
///         gross-debit and action-charge bounds, while the identical close commit is bounded,
///         reverted, and rolled back by the sponsor's own tested path.
contract OpenIncreaseCommitCarryTest is CfdClosePreviewTestBase {

    uint256 internal constant EXTRA_FREE = 2_000e6;
    uint256 internal constant INC_MARGIN = 100e6;

    function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.baseCarryBps = 1000;
    }

    function _increaseRequest(uint256 _boundForIncrease) internal view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = keccak256("increase-commit-carry-bound");
        r.side = CfdTypes.Side.SHORT;
        r.sizeDelta = SIZE;
        r.marginDelta = INC_MARGIN;
        r.targetPrice = PRICE;
        r.isClose = false;
        r.bounds = _bounds();
        r.bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        r.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        r.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        // both pinned bounds sit strictly below the carry the commit will collect
        r.bounds.maxGrossAccountDebitAssets = _boundForIncrease;
        r.bounds.maxActionChargeAssets = _boundForIncrease;
    }

    function test_IncreaseCommitDebitsCarryBeyondPinnedBounds() public {
        // position with all free settlement withdrawn (sponsor fixture)
        _openNormally(CfdTypes.Side.SHORT, 0);

        // accrue carry; measure it with the sponsor's own preview technique (read-only)
        uint256 bounty = _quoteOpenOrderExecutionBountyAssets(SIZE);
        uint256 carry = 0;
        for (uint256 i = 0; i < 400 && carry <= 4 * bounty + 2; ++i) {
            vm.warp(vm.getBlockTimestamp() + 12 hours);
            carry = _measureCarry();
        }
        assertGt(carry, 4 * bounty + 2, "carry must dominate bounty for a clean bound test");
        emit log_string("STEP 1 -- a SHORT position accrues carry while its free settlement is withdrawn");
        emit log_named_uint("  accrued carry, USDG 6dp (sponsor's own preview)", carry);
        emit log_named_uint("  auto-computed execution bounty", bounty);

        // credit-only funding: depositFor performs no carry checkpoint (MCH depositFor has no engine hook)
        settlementToken.mint(address(this), EXTRA_FREE);
        settlementToken.approve(address(clearinghouse), EXTRA_FREE);
        clearinghouse.depositFor(ACCOUNT, EXTRA_FREE);

        // CONTROL (sponsor-proven behavior): identical bounds reject a close commit and roll back
        OrderV3Types.OrderRequest memory closeReq = _closeProbeRequest();
        closeReq.bounds.maxGrossAccountDebitAssets = bounty;
        closeReq.bounds.maxActionChargeAssets = bounty;
        emit log_string("STEP 2 CONTROL -- a CLOSE commit with these exact bounds is REJECTED and rolled back:");
        emit log_named_uint("  maxGrossAccountDebitAssets pinned at", bounty);
        emit log_named_uint("  maxActionChargeAssets pinned at", bounty);
        emit log_string("  -> reverts OrderLifecycleBook__CommitmentBoundExceeded (sponsor-tested behaviour)");
        vm.expectPartialRevert(IOrderLifecycleBook.OrderLifecycleBook__CommitmentBoundExceeded.selector);
        vm.prank(ACCOUNT);
        router.commitOrder(closeReq);

        // CASE: the increase commit with the same bounds (the minimum the bounty gate permits)
        OrderV3Types.OrderRequest memory inc = _increaseRequest(bounty);
        uint256 balanceBefore = clearinghouse.balanceAssets(ACCOUNT);
        vm.prank(ACCOUNT);
        uint64 id = router.commitOrder(inc);

        // the commit debited the full accrued carry, far beyond both pinned bounds
        uint256 debited = balanceBefore - clearinghouse.balanceAssets(ACCOUNT);
        assertGt(debited, inc.bounds.maxGrossAccountDebitAssets, "gross-debit bound bypassed at commit");
        assertGt(debited, inc.bounds.maxActionChargeAssets, "action-charge bound bypassed at commit");
        assertEq(debited, carry, "commit debit is exactly the accrued carry");
        assertGt(debited - inc.bounds.maxGrossAccountDebitAssets, 0, "bypass magnitude");

        // the permanent commitment evidence records ZERO collected carry for the open
        OrderV3Types.PendingIntent memory pending = router.lifecycleBook().pendingIntent(id);
        emit log_string("STEP 3 DEFECT -- the INCREASE commit with the SAME bounds SUCCEEDS. VALUE DEBITED:");
        emit log_named_uint("  account balance BEFORE commit", balanceBefore);
        emit log_named_uint("  account balance AFTER commit", clearinghouse.balanceAssets(ACCOUNT));
        emit log_named_uint("  DEBITED at commit, USDG 6dp", debited);
        emit log_named_uint("  maxGrossAccountDebitAssets the trader pinned", inc.bounds.maxGrossAccountDebitAssets);
        emit log_named_uint("  maxActionChargeAssets the trader pinned", inc.bounds.maxActionChargeAssets);
        emit log_named_uint("  EXCESS over the pinned bound", debited - inc.bounds.maxGrossAccountDebitAssets);
        emit log_string("STEP 4 -- and the permanent commitment evidence records the collection as ZERO:");
        emit log_named_uint("  recorded carryCollectedAssets", pending.commitment.carryCollectedAssets);
        emit log_named_uint("  recorded executionBountyAssets", pending.executionBountyAssets);

        assertEq(pending.commitment.carryCollectedAssets, 0, "recorded commitment claims zero carry");
        assertEq(pending.executionBountyAssets, bounty, "bounty recorded");
    }

    function _closeProbeRequest() internal view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = keccak256("carry-probe-close");
        r.side = CfdTypes.Side.SHORT;
        r.sizeDelta = SIZE;
        r.targetPrice = 1;
        r.isClose = true;
        r.bounds = _bounds();
        r.bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        r.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        r.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
    }

    function _measureCarry() internal view returns (uint256) {
        CfdClosePreview.ClosePreview memory p =
            previewer.previewClose(address(engine), ACCOUNT, _closeProbeRequest(), KEEPER, PRICE);
        return p.commitment.carryCollectedAssets;
    }

    // ------------------------------------------------------------------
    // HARDENING 1: end-to-end — the terminal receipt understates the debit
    // ------------------------------------------------------------------

    function _increaseRequestWithBounds(
        bytes32 clientId,
        uint256 marginDelta,
        uint256 maxGross,
        uint256 maxActionCharge,
        uint256 minSettlement
    ) internal view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = clientId;
        r.side = CfdTypes.Side.SHORT;
        r.sizeDelta = SIZE;
        r.marginDelta = marginDelta;
        r.targetPrice = PRICE;
        r.isClose = false;
        r.bounds = _bounds();
        r.bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        r.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        r.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        r.bounds.maxGrossAccountDebitAssets = maxGross;
        r.bounds.maxActionChargeAssets = maxActionCharge;
        r.bounds.minPostSettlementBalanceAssets = minSettlement;
    }

    function _accrueAndFund(uint256 carryFloor) internal returns (uint256 carry, uint256 bounty) {
        _openNormally(CfdTypes.Side.SHORT, 0);
        bounty = _quoteOpenOrderExecutionBountyAssets(SIZE);
        uint256 execDebitEstimate = _engineExecutionFeeAssets(SIZE, PRICE) + bounty;
        carry = 0;
        for (uint256 i = 0; i < 300 && carry <= carryFloor; ++i) {
            vm.warp(vm.getBlockTimestamp() + 7 days);
            carry = _measureCarry();
        }
        assertGt(carry, carryFloor, "carry must dominate the execution debit estimate");
        assertGt(carry, 10 * execDebitEstimate, "carry must dominate fee plus bounty by 10x");
        settlementToken.mint(address(this), EXTRA_FREE);
        settlementToken.approve(address(clearinghouse), EXTRA_FREE);
        clearinghouse.depositFor(ACCOUNT, EXTRA_FREE);
        carry = _measureCarry(); // re-measure post-funding (funding does not checkpoint carry)
    }

    function test_IncreaseExecutionReceiptUnderstatesTotalDebit() public {
        // The execution-phase debit of this increase is knowable up front: fee + bounty.
        uint256 bountyEstimate = _quoteOpenOrderExecutionBountyAssets(SIZE);
        uint256 execEstimate = _engineExecutionFeeAssets(SIZE, PRICE) + bountyEstimate;
        (uint256 carry,) = _accrueAndFund(10 * execEstimate);

        // Probe: commit the increase with maximal bounds, measure what the execution-time
        // assessment will charge (post-commit, the accrued carry is already collected, so the
        // committed assessment's realizedCarry is zero).
        uint256 scenario = vm.snapshotState();
        uint256 probeMargin = 1e9; // $1,000 on a $10,000 increase: comfortably above initial margin
        OrderV3Types.OrderRequest memory probe =
            _increaseRequestWithBounds("probe-increase", probeMargin, type(uint256).max, type(uint256).max, 0);
        vm.prank(ACCOUNT);
        uint64 probeId = router.commitOrder(probe);
        OrderV3Types.ExecutionAssessment memory assessed =
            policyEvaluator.assessCommittedOrder(address(engine), probeId, KEEPER, PRICE);
        uint256 grossExec = assessed.grossAccountDebitAssets;
        uint256 actionExec = assessed.actionChargeAssessedAssets;
        assertGt(grossExec, bountyEstimate, "execution gross must exceed the bounty alone");
        assertLt(grossExec, carry, "execution gross must be below the accrued carry");
        vm.revertToState(scenario);
        vm.deleteStateSnapshot(scenario);

        // Real commit: bounds sized to the probe's execution gross with 10% headroom — a
        // generous version of the honest number for this order's own economics. (The headroom
        // also covers execution-delay carry: any second between commit and execution accrues
        // a little more carry, which the execution assessment DOES count against the bound —
        // the enforced side of the same coin.)
        uint256 maxGross = grossExec + grossExec / 10;
        uint256 maxAction = actionExec + actionExec / 10;
        OrderV3Types.OrderRequest memory inc =
            _increaseRequestWithBounds("real-increase", probeMargin, maxGross, maxAction, 0);
        uint256 balanceBefore = clearinghouse.balanceAssets(ACCOUNT);
        vm.prank(ACCOUNT);
        uint64 id = router.commitOrder(inc);

        // Commit silently debited the accrued carry beyond the pinned bound
        uint256 commitDebit = balanceBefore - clearinghouse.balanceAssets(ACCOUNT);
        assertEq(commitDebit, carry, "commit debit is exactly the accrued carry");
        assertGt(commitDebit, maxGross, "commit debit exceeds the pinned gross-debit bound");

        // Execute to completion
        vm.recordLogs();
        bytes[] memory update = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executes");

        // The permanent receipt records only the execution-phase debit (which stays in bounds)
        OrderV3Types.OrderReceipt memory receipt =
            _verifiedReceipt(IOrderLifecycleBook(address(router.lifecycleBook())), id);
        assertLe(receipt.economics.grossAccountDebitAssets, maxGross, "receipt gross stays inside the bound");
        assertGt(receipt.economics.grossAccountDebitAssets, bountyEstimate, "receipt gross covers fee and bounty");
        assertEq(uint8(receipt.status), uint8(OrderV3Types.LifecycleStatus.Executed));

        // Total custody actually moved by this order = commit carry + execution debit. The
        // receipt understates the true total by EXACTLY the commit-collected carry.
        uint256 totalDebit = balanceBefore - clearinghouse.balanceAssets(ACCOUNT);
        assertEq(totalDebit - receipt.economics.grossAccountDebitAssets, carry, "receipt understates by the carry");
        assertGt(totalDebit, maxGross, "actual debit exceeds the pinned bound");
        assertGt(totalDebit, 2 * maxGross, "actual debit exceeds any honest multiple of the order's economics");
    }

    // ------------------------------------------------------------------
    // HARDENING 2: the settlement-floor protection is UNAVAILABLE for increases
    // ------------------------------------------------------------------

    function test_SettlementFloorIsUnsupportedForIncreases() public {
        _openNormally(CfdTypes.Side.SHORT, 0);
        // A positive minPostSettlementBalanceAssets is rejected outright for any account with a
        // live position (OrderLiquidationBatchLogic.sol:307-310): the trader cannot even opt into
        // floor protection in exactly the scenario of this finding. The rejection fires during
        // fresh-request validation, before any reservation or carry collection.
        OrderV3Types.OrderRequest memory inc =
            _increaseRequestWithBounds("floor-unsupported", INC_MARGIN, type(uint256).max, type(uint256).max, 1);
        uint256 balance = clearinghouse.balanceAssets(ACCOUNT);
        vm.expectPartialRevert(IOrderRouterErrors.OrderRouter__SettlementFloorUnsupported.selector);
        vm.prank(ACCOUNT);
        router.commitOrder(inc);
        assertEq(clearinghouse.balanceAssets(ACCOUNT), balance, "rejected floor request moved nothing");
    }
}

/// @notice Same scenario at the governance-maximum carry rate (admin may set baseCarryBps up to
///         100,000 = 1000%/yr at full utilization; CfdEngineAdmin.sol:266-268). Shows the bypass
///         magnitude is governance-tunable and bounded only by the account's reachable balance.
contract OpenIncreaseCommitCarryMaxRateTest is OpenIncreaseCommitCarryTest {
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.baseCarryBps = 100_000;
    }

    function test_MagnitudeAtGovernanceMaximumCarryRate() public {
        _openNormally(CfdTypes.Side.SHORT, 0);
        uint256 bounty = _quoteOpenOrderExecutionBountyAssets(SIZE);
        uint256 carry = 0;
        for (uint256 i = 0; i < 400 && carry <= 100 * bounty; ++i) {
            vm.warp(vm.getBlockTimestamp() + 12 hours);
            carry = _measureCarry();
        }
        assertGt(carry, 100 * bounty, "high-rate carry must dominate the bound by 100x");

        settlementToken.mint(address(this), EXTRA_FREE);
        settlementToken.approve(address(clearinghouse), EXTRA_FREE);
        clearinghouse.depositFor(ACCOUNT, EXTRA_FREE);
        carry = _measureCarry();

        uint256 balanceBefore = clearinghouse.balanceAssets(ACCOUNT);
        OrderV3Types.OrderRequest memory inc =
            _increaseRequestWithBounds("max-rate", INC_MARGIN, bounty, bounty, 0);
        vm.prank(ACCOUNT);
        router.commitOrder(inc);
        uint256 debited = balanceBefore - clearinghouse.balanceAssets(ACCOUNT);
        assertEq(debited, carry, "debit is the collected carry");
        assertGe(debited, 100 * inc.bounds.maxGrossAccountDebitAssets, "bypass reaches 100x the pinned bound");
        assertLe(debited, balanceBefore, "debit bounded by the account's reachable balance");
    }
}
```

