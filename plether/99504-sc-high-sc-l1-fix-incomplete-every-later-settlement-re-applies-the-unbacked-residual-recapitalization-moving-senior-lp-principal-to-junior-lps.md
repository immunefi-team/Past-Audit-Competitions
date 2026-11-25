# \#99504 [SC-High] SC-L1 fix incomplete: every later settlement re-applies the unbacked residual recapitalization, moving Senior LP principal to Junior LPs

**Submitted on Nov 13th 2026 at 22:42:40 UTC by @Lester for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99504
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

After a terminal-deficit wipe, the owner's documented `CfdEngine.recapitalizePool` restores Senior LPs through a pending bucket. The SC-L1 fix caps that bucket to what is "actually settleable" and leaves the rest pending until it is backed. The cap is checked against the whole `distributable`, and part of that already backs the principal the previous pass created. So every later reconciliation applies the unbacked residual again. Any Junior holder can trigger this with a full-balance `requestRedeem` followed by the permissionless `settleLpEpoch`. No value arrives and no price moves, yet Senior principal moves to Junior and Junior holders can withdraw it.

- Impact: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
- Severity: Critical (also met: Theft of unclaimed yield, High; floor: Contract fails to deliver promised returns, Low)
- Actor needs: no role; Junior shares bought before the wipe, any size for the queue item; one permissionless call. State: a pool after a terminal-deficit wipe whose owner recapitalization is not fully settled at the next settlement.
- Funds at risk: Senior principal restored by the owner. In the project's own mixed-inflow test, one settlement takes Senior from 25,000 to 15,000 at once, and a 10,000 Senior LP loses 9,090.90. An entity holding Junior realises **+9,017.75 USDG** in the next epoch with no new backing; on the code with the one-line fix it gets 0. If the trader-claim liability ends after its settlement, it gets **+22,518.02**, and +44,535.80 in the project's SC-L1 test state.
- Reproduce: `FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/ResidualRecapReapplied.t.sol' -vv` (file in the PoC, pinned a82d4fefb02768e5b3bddcc97dee371eee24fea8; 11 tests pass)

## Vulnerability Details

### Spec vs code

| What the project says | What happens (PoC, pinned a82d4fef) |
|---|---|
| SC-L1 response (Fixed in #29): "Anything that can't be settled yet stays pending, stays reserved everywhere, and gets applied later once it's backed." | One settlement with no new backing applies the residual: pending recap 15,000 -> 0 (mixed test), 50,040 -> 80 (SC-L1 test). |
| PR #122 invariant: "Partial settlement retains residual buckets and their reserves; repeated reconciliation cannot double-count inflows." MED-12 context there: "Reconciliation batching could change the resulting ownership." | One extra reconciliation changes the final ownership: Senior 15,000 vs 40,000 (mixed), 50,040 vs 100,000 (SC-L1). |
| `HousePoolWaterfall.t.sol:201` "Residual recap should preserve senior's recovery target" (`seniorHighWaterMark == 100,000`) | Fails after one ordinary settlement with "50040000000 != 100000000000" (the PoC catches the project's own assertion). |
| `HousePoolWaterfall.t.sol:217` "Residual recap should settle once backing becomes available" (`seniorPrincipal == 100,000`) | Fails: Senior 50,040, Junior 49,880, unassigned 80. |
| `HousePoolSnapshotParity.t.sol:155-169` `test_PartialMixedInflowsPreserveFullIntentAndSettleResidualLater`: 25,000 / 0 after one pass, 40,000 / 15,000 after the release | One settlement in between: 15,000 / 10,000 at once, 15,000 / 40,000 after the release. |
| `ACCOUNTING_SPEC.md:337` "Losses do not reduce the Senior high-water mark"; `:340` "Reserving them in step 2 prevents counting the same inflow twice." | No value changes, yet the mark falls 100,000 -> 50,040; the same cash is reserved for the bucket and also backs the principal. |
| `HousePoolPendingPreviewLib.sol:107` "recapitalization never accrues to junior principal"; Cecuro MED-12 comment: "partial-settlement residuals stay reserved" | 49,880 (SC-L1 test) and 30,000 (exact-gap test) of the owner's recapitalization end as Junior principal. |

### Root cause

`packages/perps/src/HousePool.sol:1829-1834` (added by the SC-L1 fix, 31a38bb7):

```solidity
function _settleablePendingClaimantBuckets(
    HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal pure returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets) {
    return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
}
```

The cap is the whole `distributable`. On the first pass that is correct: both principals are zero, so all of it is free. On every later pass, part of `distributable` already backs `seniorPrincipal + juniorPrincipal` from the previous pass. `HousePoolReconcilePlanLib.planReconcile` (`:68-71`) then subtracts the settleable amount from `distributable` ("Settleable pending claimant assets already represented in `snapshot.distributable` and therefore reserved", `:41-42`). It finds the principal unbacked, and `absorbLoss` charges it. `_applyPendingClaimantBucketsLive` (`HousePool.sol:1759-1776`) credits the amount again and decrements the bucket. When the loss took both principals to zero, `applyClaimantRecapitalizationIntent` (`HousePoolPendingPreviewLib.sol:117-122`) takes the empty-bootstrap branch and sets `seniorHighWaterMark = max(remaining bucket, amount)`. Settleable revenue then goes to Junior, because the new mark leaves no Senior gap.

Project numbers (SC-L1 test), second pass, no value change (pool cash 100,000, trader claim 50,040, `distributable` 49,960):

| step | Senior principal | high-water mark | pending recap |
|---|---|---|---|
| after the project's first reconcile | 49,960 | 100,000 | 50,040 |
| cap: min(50,040, 49,960) = 49,960 reserved; plan sees 0 for 49,960 of principal -> loss | 0 | 100,000 | 50,040 |
| empty bootstrap: principal += 49,960, mark = max(50,040, 49,960) | 49,960 | **50,040** | **80** |

### Trigger (ordinary calls only)

- `HousePool.reconcile()` is `onlyVault`, and no vault function calls it. The production path is `settleLpEpoch`. `HousePool.settleLpEpoch` is permissionless with no open positions or a frozen oracle (`HousePool.sol:794-820`). With open positions it is reached through `OrderRouter.settleLpEpoch(bytes[])`, "Permissionless and available while the router admin is paused".
- A settlement must advance a queue item; with nothing queued it reverts `HousePool__NoLpEpochProgress` (asserted). A Junior holder's full-balance `requestRedeem` has no minimum size. At zero Junior principal it is refunded (`HousePool.sol:960-985`), and that counts as progress (`juniorProcessedEpochs == 1` in the PoC). So a Junior holder can trigger a settlement whenever it likes. Keepers that settle every epoch trigger it anyway.
- The state comes from the owner's documented recovery action. `SCOPE.md` trusted roles: "recapitalize the pool". `SECURITY.md`: "Owner-funded recapitalization". Runbook: "4. Recapitalize or clear positions. 5. Clear degraded mode only after raw solvency `E >= L` is genuinely restored". Revenue is queued in this state by design: `recordClaimantInflow` NatSpec says "revenue is explicitly queued when both stored tranche principals are zero".

## Impact Details

### Immediate form and actor profit (lead)

Base: the project's mixed test (wipe, 40,000 owner recapitalization + 15,000 revenue, 30,000 trader claim, one reconcile: Senior 25,000, mark 40,000, residual 15,000). Holders: a 10,000 Senior LP, a passive 10,000 Junior LP, and one entity with two Junior accounts (100,000 and 1,000). The entity bought them by ordinary deposits **before** the wipe. After the wipe, Junior deposits revert (asserted), as documented: "a wiped tranche cannot be silently revived by an ordinary asynchronous deposit" (`ACCOUNTING_SPEC.md:326`).

| test | what happens | result (USDG) |
|---|---|---|
| `test_EntityProfit_MixedState` | T1: the small account requests its full balance and the entity itself calls `settleLpEpoch`. T2 (next epoch): the large account redeems. No new backing. | At T1: Senior 25,000 -> 15,000, residual 15,000 -> 0, Senior LP -9,090.90, entity value 0 -> 9,017.87. At T2 the entity **realises 9,017.750652**. Nobody settling: 0. Fixed code, same calls: 0 (both requests refunded). |
| same test, liability ends after T1 | The project's own release step (trader claim to 0, no new cash) comes after the entity's T1 settlement. | Entity realises 36,044.83 vs 13,526.81 without the T1 settlement: **+22,518.017728**. The latter equals the project's expected final state and the fixed code. Senior -24,999.86; Senior LP -22,727.13. |
| `test_EntityProfit_ProjectState` | Same calls on the project's SC-L1 test state | Without backing nothing is realisable in any arm (Junior stays 0). Liability ends after the entity's settlement: **+44,535.797448** vs 0; Senior -49,960; Senior LP -45,418.14. |
| `test_MixedInflows_oneOrdinarySettlement_movesSeniorToJunior` | Mixed test + one settlement triggered by the Junior LP's full-balance request, any address calling | At once: Senior 25,000 -> 15,000; Junior LP paid **9,090.999660** USDG out of pool cash in the same transaction; Senior LP -9,090.901544. After the release: Senior 15,000 / Junior 40,000 vs the project's 40,000 / 15,000. |

Pool cash plus USDG paid out is conserved in every arm. The party values add up to principals plus payouts. The entity puts no USDG in after the wipe, and its pre-wipe deposits are lost in every arm alike.

Baseline, stated plainly: every ordinary settlement in that window moves the residual. Against an arm where the entity's own later redemption settlement runs anyway, the extra call changes its result by -0.12. An entity holding Junior gains by making sure one settlement happens, measured against nobody settling and against the code with the one-line fix. Value is conserved: what Senior holders lose, Junior holders gain.

### Recovery target (high-water mark) decline

| test | result |
|---|---|
| `test_ProjectTest_withOneOrdinarySettlement` (SC-L1 test + one settlement) | Pool cash 100,000, trader claim 50,040 and Senior 49,960 are unchanged, yet the mark goes 100,000 -> 50,040 and pending 50,040 -> 80. The project's assertions at `:201` and `:217` fail with their exact messages. Final: Senior 50,040, Junior 49,880, unassigned 80. |
| `test_Control_noSettlement_projectAssertionsHold` | Same clock, no added call: the project's assertions hold (Senior 100,000, Junior 0). |
| `test_RunMinusControl_perParty` | Senior -49,960 (Senior LP -45,418.144113); Junior +49,880 (Junior LP +45,345.906306); unassigned +80; pool cash equal. |
| `test_ExactGapRecap_routineSettlements_vs_none` | The owner recapitalizes exactly the 50,000 deficit and the backing arrives in ten steps. Routine settlements after steps 1-4: mark 50,000 -> 20,000; final Senior 20,000 / Junior 30,000 vs 50,000 / 0. |

### Whole state from transactions

`ResidualRecapFromTransactionsTest` builds the wipe from trades and mark pushes (no `vm.store`, no `stdstore`, no prank as a protocol contract). The owner calls only `recapitalizePool` and `clearDegradedMode`, and `degradedMode` is asserted after every step. Each extra keeper settlement moves 94.347204 (the revenue bucket) out of Senior's recovery. With 59 extra settlements over 120 epochs, a 50,000 Senior LP ends with 1,644.14 instead of 7,101.40: Senior holders -5,566.55, Junior holders +11,133.03.

Owner-procedure step: the trigger works only after `clearDegradedMode`. Before it, every settlement reverts `HousePool__DegradedMode`, and the clear reverts `CfdEngine__StillInsolvent` until the whole gap is paid. After an exact-gap recapitalization and the clear, the recapitalization is still entirely unsettled, so the state exists. The immediate 25,000 -> 15,000 form is not reproduced from transactions; it uses the project's own test construction.

## Severity (program rules)

- **Row.** Senior LP principal, backed by pool cash, moves to Junior holders, who withdraw it as USDG (9,090.999660 in the same transaction). No price moves and no value arrives. That is user funds taken, so it is the Critical row. The Senior recovery owed from later backing also goes to Junior (High "Theft of unclaimed yield", tests above). With the fix the same calls leave Senior intact.
- **Actor profit (`SCOPE.md:239`).** The griefing rule applies "where the attacker gains no profit". Here the triggering entity realises +9,017.75 USDG in the next epoch for the cost of gas, against nobody settling and against the fixed code. It realises +22,518.02 or +44,535.80 when the liability ends after its settlement. The baseline is stated above: an ordinary settlement by anyone in that window would also move the residual.
- **Financial risk (`:235`).** The actor puts nothing at risk after the wipe beyond gas. Its Junior shares are worth 0 in every arm until a settlement moves the residual.
- **Feasibility (`:336`).** The trigger has no obstacle: no minimum request size, a permissionless call, any epoch. The rare part is the state, given as a fact below.
- **Role (`:196`, `:251`).** The owner performs only its documented recovery: `recapitalizePool`, then `clearDegradedMode` once `E >= L`. That is "a legitimate, documented action by a trusted role that triggers a bug", and "an in-scope impact arises from a logic error that a privileged address can trigger, which remains valid". The harmful call is not privileged.
- **Keeper (`:200`).** "Keepers are not trusted ... harm caused by any caller acting as a keeper is in scope." The trigger is a keeper call anyone can make.
- **Rare state, as a fact.** It needs a terminal-deficit wipe (both tranches at zero with shares outstanding) and an owner recapitalization not fully settled when the next settlement runs. In a trade-built wipe, settlements stay blocked until the owner clears degraded mode; after the clear the recapitalization is still unsettled (test above). The mixed form needs a revenue bucket too, which this state queues by design. Each pass moves at most the part of the bucket that `distributable` appears to back.

## Relationship to published known issues

- **SC-L1 (pre-audit, Fixed in #29).** Same code, same numbers (100,000 / 49,960). The exercised line, `HousePool.sol:1833`, is the fix's own cap (31a38bb7). The fix holds for one reconcile and fails on the next. The program counts "the fix at commit a82d4fef is incomplete" as valid.
- **Cecuro MED-12 (Addressed by PR #122).** Its fix kept "partial-settlement limits" unchanged and states "partial-settlement residuals stay reserved". The immediate form is its own mixed case with one settlement added.
- **Not claimed:** the FIRST reconcile's reset of the high-water mark to the recapitalization amount. It is documented ("With senior owners and zero claimed principal ... sets the high-water mark to `max(recapitalizationTargetAssets, amount)`"; test "Explicit recapitalization should reset the HWM"). Every test here keeps the project's values for that first pass and starts after it. This is not an entry by deposit at near-zero NAV after a wipe: there is no deposit and no price move. Also not claimed: settlement work spent on zero-payout epochs (MED-24, `SECURITY.md` Known Limitations), and multi-pass rounding of redemption funding (`SECURITY.md:1411`).
- AI scan, collaborative audit, Cantina: no item on pending claimant buckets or a second reconciliation.

## Recommended fix

Cap the settleable buckets by the value that does not already back claimed principal, at both call sites (`HousePool.sol:1566` live, `:1619` preview):

```solidity
uint256 claimed = seniorPrincipal + juniorPrincipal;
uint256 free = reconcileSnapshot.distributable > claimed ? reconcileSnapshot.distributable - claimed : 0;
settleableClaimantBuckets = HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, free);
```

With this change applied to a copy, all eight bug tests in the PoC fail on their first bug assertion: Senior stays at 25,000, the mark stays at 100,000 or 50,000, and the high-water mark is equal in RUN and CONTROL in the transactions-only test. The control, the deposit/queue check and the owner-procedure gates still pass. A regression test: run the project's two partial-settlement tests with one `settleLpEpoch` between the first reconcile and the release.

## References

- Root cause: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834
- Plan reserve: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L71
- Empty bootstrap: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L135
- Live bucket step: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1759-L1776
- Permissionless settlement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L820
- Project tests: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol#L177-L220 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/differential/HousePoolSnapshotParity.t.sol#L155-L169
- Spec: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L326-L340
- SC-L1 and its response: `audits/plether-pre-audit-report.pdf` (SC-L1, "Fixed in Plether-Fi/plether-core#29"); PR #122: https://github.com/Plether-Fi/plether-core/pull/122


## Proof of Concept

## Proof of Concept

One Foundry file, pinned commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, with two contracts on the project's own test bases. The full file is at the end.

### Run

```
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
# save the file below as packages/perps/test/ResidualRecapReapplied.t.sol
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/ResidualRecapReapplied.t.sol' -vv
```

Checked on a clean copy of the pinned tree, cold build (forge 1.5.1-stable, solc 0.8.35): 11 passed, 0 failed; also with `--isolate`. Each bug test asserts the observed values next to the project's expected values, so it passes while the bug is present.

### What is built how

- `ResidualRecapReappliedTest` (`HousePoolTestBase`): the states are the project's two tests of a partial settlement (`HousePoolSnapshotParity.t.sol:155-169`, `HousePoolWaterfall.t.sol:177-220`), built as they build them: cash burned to wipe both tranches, trader-claim liability set with `stdstore`, revenue recorded as the engine. One change: the recapitalization goes through the owner's `CfdEngine.recapitalizePool`. Every added step is ordinary calls only: a Junior holder's full-balance `requestRedeem` and a permissionless `settleLpEpoch`. RUN and CONTROL arms share one snapshot and one clock. `degradedMode` is asserted false around every settlement.
- `ResidualRecapFromTransactionsTest` (`BasePerpTest`): the whole state from transactions. Wallets are funded once at setUp; LP deposits go through `requestDeposit` / keeper settlement / `claimDeposit`; trades go through `OrderRouter` with keeper execution and MockPyth prices; the owner calls only `recapitalizePool` and `clearDegradedMode`. No `vm.store`, no `stdstore`, no prank as a protocol contract. `degradedMode` is asserted after every step, and every refused call by its selector.

### Tests

1. `test_EntityProfit_MixedState`: one entity (100,000 + 1,000 Junior, bought before the wipe). Its small account requests its full balance, the entity settles itself, and its large account redeems next epoch. T1: Senior 25,000 -> 15,000, Senior LP -9,090.901544, entity value 9,017.874062. T2: the entity realises 9,017.750652 USDG with no backing; nobody settling: 0; against an arm in which its own T2 settlement runs anyway: -0.123410. Liability ends after its T1 settlement: 36,044.828822 vs 13,526.811094 (project expectation), +22,518.017728; Senior LP -22,727.129327. Conservation asserted in every arm.
2. `test_EntityProfit_ProjectState`: same on the SC-L1 test state. No backing: 0 in every arm, mark 100,000 -> 50,040. Liability ends after its settlement: +44,535.797448 vs 0; Senior -49,960.
3. `test_MixedInflows_oneOrdinarySettlement_movesSeniorToJunior`: the project's mixed test + one settlement (Junior LP's request, any address settles), no value arriving. Senior 25,000 -> 15,000, residual 15,000 -> 0, Junior LP paid 9,090.999660 USDG out of pool cash, Senior LP -9,090.901544. After the release: 15,000 / 40,000 vs the project's 40,000 / 15,000.
4. `test_ProjectTest_withOneOrdinarySettlement`: pool cash, claim and Senior unchanged; mark 100,000 -> 50,040, pending 50,040 -> 80. The project's own assertion helpers fail with "Residual recap should preserve senior's recovery target: 50040000000 != 100000000000" and "Residual recap should settle once backing becomes available: 50040000000 != 100000000000". Final 50,040 / 49,880 / unassigned 80.
5. `test_Control_noSettlement_projectAssertionsHold`: same clock, no call; the project's assertions hold.
6. `test_RunMinusControl_perParty`: Senior -49,960 (Senior LP -45,418.144113), Junior +49,880 (Junior LP +45,345.906306), unassigned +80, pool cash equal.
7. `test_ExactGapRecap_routineSettlements_vs_none`: exact 50,000 recapitalization, backing in ten steps, settlements after steps 1-4: mark 50,000 -> 20,000; final 20,000 / 30,000 vs 50,000 / 0 (preview checked against a real settlement).
8. `test_NoDepositEntry_and_settlementNeedsAQueuedRequest`: Junior `requestDeposit` reverts after the wipe (`TerminallyWiped`) and after the recapitalization; a settlement with nothing queued reverts `HousePool__NoLpEpochProgress`.
9. `test_FromTransactions_ownerProcedureGates`: wipe by trades (`degradedMode` latches at a 0.985 close; a SHORT closes at the 2.00 cap). Settlements revert `HousePool__DegradedMode` before any owner action, after half the gap and after the full gap; `clearDegradedMode` reverts `CfdEngine__StillInsolvent` until the gap (7,243.528343) is paid. After the clear, the first settlement succeeds with both principals 0 and the whole recapitalization still pending; a trader's margin deposit queues a 94.347204 revenue bucket.
10. `test_FromTransactions_oneExtraSettlement` (n = 2) and 11. `test_FromTransactions_59ExtraSettlements` (n = 60, 120 epochs): RUN settles at the cap until n settlements succeed, CONTROL settles once, same epochs and holder calls; then the same unwind and full redemption. Each extra settlement lowers the Senior mark, the pending recapitalization and unassigned by one bucket (asserted). n = 60: Senior LP 7,101.403002 -> 1,644.138024, Senior holders -5,566.545883, Junior holders +11,133.030918 (trigger +3,336.556720); traders, keeper, treasury unchanged; every USDG at a listed address; party sum equal within 2 atoms.

Test-quality control: with the fix proposed in the description applied to a copy (two lines, `HousePool.sol:1566` and `:1619`), tests 1-4, 6, 7, 10 and 11 fail on their first bug assertion (for example "T1: senior principal 25,000 -> 15,000 in the entity's settlement: 25000000000 != 15000000000"; "Senior HWM falls one bucket per extra settlement: 0 != 5566485036"). Tests 5, 8 and 9 pass. The source was restored and `diff -r` against the pinned tree is clean.

### Output excerpt (`-vv`)

```
[PASS] test_EntityProfit_MixedState()
     ENTITY USDG  RUN - nobody settles      = + 9017750652
     ENTITY USDG  RUN - CONTROL             = - 123410
     ENTITY USDG  RUN-hold - CONTROL-hold   = + 22518017728
[PASS] test_FromTransactions_59ExtraSettlements()
  cap: Senior HWM RUN / CONTROL 1677043307 7243528343
     SEN_A (Senior LP) | 1644138024 | 7101403002 | -5457264978
     JUN_B (Junior LP, trigger) | 238808053393 | 235471496673 | +3336556720
  Senior holders RUN - CONTROL -5566545883
  Junior holders RUN - CONTROL +11133030918
Suite result: ok. 8 passed; 0 failed; 0 skipped
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

### File: `packages/perps/test/ResidualRecapReapplied.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// A residual pending recapitalization is applied again by every later reconciliation: the settleable cap is the whole
// distributable (HousePool._settleablePendingClaimantBuckets, HousePool.sol:1829-1834), part of which already backs the
// principal the previous pass created.
// Pinned source: immunefi-team/audit-comp-plether a82d4fefb02768e5b3bddcc97dee371eee24fea8.
// Save as packages/perps/test/ResidualRecapReapplied.t.sol; from the repository root:
//   FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/ResidualRecapReapplied.t.sol' -vv
//
// Contract 1 (project's HousePoolTestBase): the states are the project's own two tests of a partial pending-bucket
// settlement (HousePoolSnapshotParity.t.sol:155-169, HousePoolWaterfall.t.sol:177-220), built as they build them (cash
// burned to wipe both tranches, trader-claim liability written with stdstore, revenue recorded as the engine); the
// recapitalization goes through the owner's CfdEngine.recapitalizePool. Every added step is ordinary calls only:
// requestRedeem of a Junior balance and the permissionless HousePool.settleLpEpoch (no open positions; with open
// positions the same call is OrderRouter.settleLpEpoch). Tests assert the observed values next to the project's.
// Contract 2 (project's BasePerpTest): the whole state from transactions, degradedMode asserted after every step.

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {HousePoolTestBase} from "./perps/shared/HousePoolTestBase.sol";
import {console} from "forge-std/console.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract ResidualRecapReappliedTest is HousePoolTestBase {

    address internal constant SEN_LP = address(0x5E01); // ordinary Senior LP (10,000)
    address internal constant JUN_LP = address(0x1E01); // ordinary Junior LP (10,000)
    address internal constant ANYONE = address(0xBEEF); // any address acting as keeper
    address internal constant E_MAIN = address(0xE1); // one entity: large Junior holder (100,000), calls the settlement
    address internal constant E_SMALL = address(0xE2); // same entity: small Junior holder (1,000), the queue item

    uint256 internal constant PENDING = 100_000e6; // the project's numbers, HousePoolWaterfall.t.sol:182-184
    uint256 internal constant CLAIM = 50_040e6;
    uint256 internal constant SETTLEABLE = PENDING - CLAIM; // 49,960

    struct Book {
        uint256 seniorPrincipal;
        uint256 juniorPrincipal;
        uint256 hwm;
        uint256 pendingRecap;
        uint256 unassigned;
        uint256 poolRaw;
        // party value = pro-rata principal of shares held (incl. shares in a pending request) + USDG received
        uint256 senLp;
        uint256 senSeed;
        uint256 junLp;
        uint256 junSeed;
        uint256 junLpUsdg;
        uint256 eMain;
        uint256 eSmall;
        uint256 eUsdg; // USDG received by the entity (E_MAIN + E_SMALL)
    }

    uint256 internal reqJun;
    uint256 internal reqSmall;
    uint256 internal reqMain;

    function setUp() public override {
        super.setUp();
        _fundSenior(SEN_LP, 10_000e6);
        _fundJunior(JUN_LP, 10_000e6);
    }

    // ------------------------------------------------------------------------------------------------ helpers

    /// @dev Both entity accounts buy Junior by ordinary deposits BEFORE the wipe (after it Junior deposits revert).
    function _fundEntity() internal {
        _fundJunior(E_MAIN, 100_000e6);
        _fundJunior(E_SMALL, 1_000e6);
    }

    function _noDegraded() internal view {
        assertFalse(engine.degradedMode(), "degradedMode false");
    }

    /// @dev The engine owner's documented recovery call (owner = this test contract, as in the project's base).
    function _ownerRecapitalizes(
        uint256 amount
    ) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
        assertEq(pool.pendingRecapitalizationAssets(), amount, "recapitalization queued for normal reconciliation");
    }

    function _wipe() internal {
        settlementToken.burn(address(pool), pool.totalAssets());
        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 0, "both tranches wiped");
        assertGt(juniorVault.totalSupply(), 0, "Junior shares outstanding");
        _noDegraded();
    }

    /// @dev The project's partial mixed-inflow test (HousePoolSnapshotParity.t.sol:155-169) up to its first reconcile,
    ///      with its expected values, then 1 hour + 1 s (deposit cooldown).
    function _mixedState() internal {
        _wipe();
        _ownerRecapitalizes(40_000e6);
        settlementToken.mint(address(pool), 15_000e6);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            15_000e6, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
        );
        _setTotalTraderClaim(30_000e6);
        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), 25_000e6, "project: senior 25,000");
        assertEq(pool.juniorPrincipal(), 0, "project: junior 0");
        assertEq(pool.seniorHighWaterMark(), 40_000e6, "project: HWM 40,000");
        assertEq(pool.pendingRecapitalizationAssets(), 15_000e6, "project: residual 15,000");
        _noDegraded();
        vm.warp(block.timestamp + 1 hours + 1);
    }

    /// @dev The project's SC-L1 regression test (HousePoolWaterfall.t.sol:178-203) up to its first reconcile, with its
    ///      own assertions, then 1 hour + 1 s.
    function _projectState() internal {
        _wipe();
        _ownerRecapitalizes(PENDING);
        _setTotalTraderClaim(CLAIM);
        vm.prank(address(seniorVault));
        pool.reconcile();
        this.projectAssertionsAfterFirstReconcile();
        _noDegraded();
        vm.warp(block.timestamp + 1 hours + 1);
    }

    /// @dev The project's release step: the trader-claim liability ends (no new cash) and the pool reconciles.
    function _release(
        address vault
    ) internal {
        _setTotalTraderClaim(0);
        vm.prank(vault);
        pool.reconcile();
    }

    function _pro(
        uint256 principal,
        uint256 shares,
        uint256 supply
    ) internal pure returns (uint256) {
        return supply == 0 ? 0 : (principal * shares) / supply;
    }

    function _jShares(
        address who,
        uint256 req
    ) internal view returns (uint256 s) {
        s = juniorVault.balanceOf(who);
        if (req != 0) {
            s += juniorVault.pendingRedeemRequest(req, who);
        }
    }

    function _book() internal view returns (Book memory b) {
        b.seniorPrincipal = pool.seniorPrincipal();
        b.juniorPrincipal = pool.juniorPrincipal();
        b.hwm = pool.seniorHighWaterMark();
        b.pendingRecap = pool.pendingRecapitalizationAssets();
        b.unassigned = pool.unassignedAssets();
        b.poolRaw = settlementToken.balanceOf(address(pool));
        uint256 sSup = seniorVault.totalSupply();
        uint256 jSup = juniorVault.totalSupply();
        b.senLp = _pro(b.seniorPrincipal, seniorVault.balanceOf(SEN_LP), sSup);
        b.senSeed = _pro(b.seniorPrincipal, seniorVault.balanceOf(address(this)), sSup);
        b.junLpUsdg = settlementToken.balanceOf(JUN_LP);
        b.junLp = _pro(b.juniorPrincipal, _jShares(JUN_LP, reqJun), jSup) + b.junLpUsdg;
        b.junSeed = _pro(b.juniorPrincipal, juniorVault.balanceOf(address(this)), jSup);
        b.eMain = _pro(b.juniorPrincipal, _jShares(E_MAIN, reqMain), jSup) + settlementToken.balanceOf(E_MAIN);
        b.eSmall = _pro(b.juniorPrincipal, _jShares(E_SMALL, reqSmall), jSup) + settlementToken.balanceOf(E_SMALL);
        b.eUsdg = settlementToken.balanceOf(E_MAIN) + settlementToken.balanceOf(E_SMALL);
    }

    function _log(
        string memory tag,
        Book memory b
    ) internal pure {
        console.log(tag);
        console.log("   Senior principal / HWM     ", b.seniorPrincipal, b.hwm);
        console.log("   Junior principal / pool raw", b.juniorPrincipal, b.poolRaw);
        console.log("   pending recap / unassigned ", b.pendingRecap, b.unassigned);
        console.log("   SEN_LP / JUN_LP (incl. USDG)", b.senLp, b.junLp);
        console.log("   entity USDG / entity value ", b.eUsdg, b.eMain + b.eSmall);
    }

    function _delta(
        string memory name,
        uint256 run,
        uint256 control
    ) internal pure {
        if (run >= control) {
            console.log(string.concat("   ", name, " = +"), run - control);
        } else {
            console.log(string.concat("   ", name, " = -"), control - run);
        }
    }

    function _requestAll(
        address who
    ) internal returns (uint256 id) {
        uint256 shares = juniorVault.balanceOf(who);
        vm.prank(who);
        id = juniorVault.requestRedeem(shares, who, who);
    }

    /// @dev Claims the refund (zero Junior NAV) or the funded USDG of a settled request.
    function _collect(
        address who,
        uint256 id
    ) internal {
        if (juniorVault.refundableRedeemRequest(id, who) > 0) {
            vm.prank(who);
            juniorVault.claimRedeemRefund(id, who, who);
        }
        uint256 c = juniorVault.claimableRedeemRequest(id, who);
        if (c > 0) {
            vm.prank(who);
            juniorVault.claimRedeem(id, c, who, who);
        }
    }

    /// @dev Permissionless HousePool.settleLpEpoch by `caller` (no open positions).
    function _settleAs(
        address caller
    ) internal returns (IHousePool.LpEpochSettlementResult memory r) {
        assertFalse(engine.terminalNavSnapshot().hasOpenPositions, "no open positions: settleLpEpoch permissionless");
        _noDegraded();
        vm.prank(caller);
        r = pool.settleLpEpoch(0, 0);
        _noDegraded();
    }

    /// @dev The added step of the project-state tests: JUN_LP requests its whole balance, at maturity ANYONE settles;
    ///      JUN_LP collects the refund (zero Junior NAV) or the funded USDG.
    function _ordinarySettlement() internal returns (IHousePool.LpEpochSettlementResult memory r) {
        reqJun = _requestAll(JUN_LP);
        vm.warp(pool.lpEpochStart(reqJun));
        r = _settleAs(ANYONE);
        _collect(JUN_LP, reqJun);
    }

    /// @dev Same clock as _ordinarySettlement without any call (a request made now targets epoch e + 1, or e + 2 within
    ///      the last 300 seconds of the hour).
    function _sameTimeNoCall() internal {
        uint256 e = block.timestamp / 1 hours;
        uint256 target = block.timestamp < (e + 1) * 1 hours - 300 ? e + 1 : e + 2;
        vm.warp(target * 1 hours);
    }

    // ------------------------------------------------------------------------------------------------ entity arms

    /// RUN: E_SMALL requests its full balance; at maturity (T1) E_MAIN settles; (optionally the liability ends at T1);
    /// E_MAIN requests its full balance and settles in the next epoch (T2); both collect.
    function _runArm(
        bool releaseAtT1
    ) internal returns (Book memory atT1, Book memory atT2, uint256 t1, uint256 t2) {
        reqSmall = _requestAll(E_SMALL);
        t1 = pool.lpEpochStart(reqSmall);
        vm.warp(t1);
        assertEq(_settleAs(E_MAIN).juniorProcessedEpochs, 1, "RUN T1: E_SMALL's epoch processed");
        _collect(E_SMALL, reqSmall);
        atT1 = _book();
        if (releaseAtT1) {
            _setTotalTraderClaim(0);
        }
        reqMain = _requestAll(E_MAIN);
        t2 = pool.lpEpochStart(reqMain);
        assertGt(t2, t1, "E_MAIN's request matures in a later epoch");
        vm.warp(t2);
        _settleAs(E_MAIN);
        _collect(E_MAIN, reqMain);
        atT2 = _book();
    }

    /// CONTROL: no call at T1 (same clock); (optionally the liability ends at T1); at T1 both entity accounts request
    /// their full balances; E_MAIN settles at T2.
    function _controlArm(
        uint256 t1,
        bool releaseAtT1
    ) internal returns (Book memory atT2, uint256 t2) {
        vm.warp(t1);
        if (releaseAtT1) {
            _setTotalTraderClaim(0);
        }
        reqSmall = _requestAll(E_SMALL);
        reqMain = _requestAll(E_MAIN);
        assertEq(reqSmall, reqMain, "CONTROL: both requests in the same epoch");
        t2 = pool.lpEpochStart(reqMain);
        vm.warp(t2);
        _settleAs(E_MAIN);
        _collect(E_SMALL, reqSmall);
        _collect(E_MAIN, reqMain);
        atT2 = _book();
    }

    struct Arms {
        Book start;
        Book r1; // RUN at T1, right after the entity's settlement
        Book r2; // RUN at T2, no backing
        Book c2; // CONTROL at T2, no backing
        Book z2; // nobody settles up to T2
        Book rh; // RUN, liability ends at T1, at T2
        Book ch; // CONTROL, liability ends at T1, at T2
    }

    function _back(
        uint256 snap
    ) internal {
        vm.revertToState(snap);
        reqSmall = 0;
        reqMain = 0;
    }

    function _allArms(
        bool mixed
    ) internal returns (Arms memory a) {
        _fundEntity();
        if (mixed) {
            _mixedState();
        } else {
            _projectState();
        }
        uint256 snap = vm.snapshotState();
        a.start = _book();
        uint256 t1;
        uint256 t2;
        uint256 tx2;
        (a.r1, a.r2, t1, t2) = _runArm(false);
        _back(snap);
        (a.c2, tx2) = _controlArm(t1, false);
        assertEq(tx2, t2, "same T2");
        _back(snap);
        vm.warp(t2);
        a.z2 = _book();
        _back(snap);
        (,, , tx2) = _runArm(true);
        a.rh = _book();
        assertEq(tx2, t2, "same T2");
        _back(snap);
        (a.ch, tx2) = _controlArm(t1, true);
        assertEq(tx2, t2, "same T2");
        _back(snap);
        _log("START", a.start);
        _log("RUN at T1 (entity settled; no value arrived)", a.r1);
        _log("RUN at T2 (entity redeemed next epoch; no backing)", a.r2);
        _log("CONTROL at T2 (no T1 call; entity redeems at T2)", a.c2);
        _log("NOBODY SETTLES, T2", a.z2);
        _log("RUN-hold at T2 (liability ended after the entity's T1 settlement)", a.rh);
        _log("CONTROL-hold at T2 (liability ended at T1, no T1 settlement)", a.ch);
        _delta("ENTITY USDG  RUN - nobody settles     ", a.r2.eUsdg, a.z2.eUsdg);
        _delta("ENTITY USDG  RUN - CONTROL            ", a.r2.eUsdg, a.c2.eUsdg);
        _delta("ENTITY USDG  RUN-hold - CONTROL-hold  ", a.rh.eUsdg, a.ch.eUsdg);
        _delta("SEN_LP       RUN-hold - CONTROL-hold  ", a.rh.senLp, a.ch.senLp);
        _assertConserved(a.start, a.r2, false);
        _assertConserved(a.start, a.c2, false);
        _assertConserved(a.start, a.rh, true);
        _assertConserved(a.start, a.ch, true);
    }

    function _near(
        uint256 x,
        uint256 y,
        uint256 tol,
        string memory what
    ) internal pure {
        assertLe(x > y ? x - y : y - x, tol, what);
    }

    /// @dev Pool cash + entity USDG is constant; claimed value (S + J + unassigned + paid out) equals START without
    ///      backing, or all pool cash once the liability has ended; party values add up (floor rounding, 6 parties).
    function _assertConserved(
        Book memory s,
        Book memory b,
        bool released
    ) internal pure {
        assertEq(b.poolRaw + b.eUsdg, s.poolRaw, "pool cash + entity USDG constant");
        uint256 claimed = b.seniorPrincipal + b.juniorPrincipal + b.unassigned + b.eUsdg;
        _near(claimed, released ? s.poolRaw : s.seniorPrincipal + s.juniorPrincipal + s.unassigned, 1, "claimed value");
        _near(
            b.senLp + b.senSeed + b.junLp + b.junSeed + b.eMain + b.eSmall,
            b.seniorPrincipal + b.juniorPrincipal + b.eUsdg,
            6,
            "party values = principals + paid out"
        );
    }

    // ------------------------------------------------------------------------------------------------ tests

    /// ENTITY PROFIT, mixed state (the project's partial mixed-inflow test). One entity, Junior bought before the wipe.
    /// T1: E_SMALL's full-balance request + E_MAIN's own settlement: Senior 25,000 -> 15,000 at once, no value arriving.
    /// T2 (next epoch, no backing): the entity realises 9,017.750652 USDG; nobody settling, or the fixed code: 0.
    /// Against an arm in which the entity's own T2 redemption settlement runs anyway: -0.123410 (that settlement moves
    /// the residual too). If the trader-claim liability ends after the entity's T1 settlement: +22,518.017728 against
    /// the same calls without it (= the project's expected final state = the fixed code).
    function test_EntityProfit_MixedState() public {
        Arms memory a = _allArms(true);
        assertEq(a.start.seniorPrincipal, 25_000e6, "START = project: senior 25,000");
        assertEq(a.r1.seniorPrincipal, 15_000e6, "T1: senior principal 25,000 -> 15,000 in the entity's settlement");
        assertEq(a.r1.pendingRecap, 0, "T1: residual recapitalization consumed without backing");
        assertEq(a.r1.eMain + a.r1.eSmall, 9_017_874_062, "T1: entity value 0 -> 9,017.874062");
        assertEq(a.start.senLp - a.r1.senLp, 9_090_901_544, "T1: SEN_LP -9,090.901544");
        assertEq(a.r2.eUsdg, 9_017_750_652, "T2: entity realises 9,017.750652 USDG without backing");
        assertEq(a.z2.eUsdg, 0, "nobody settles: entity holds Junior shares worth 0");
        assertEq(a.z2.seniorPrincipal - a.r2.seniorPrincipal, 9_999_863_014, "Senior -9,999.863014 vs nobody settling");
        assertEq(a.c2.eUsdg - a.r2.eUsdg, 123_410, "vs CONTROL (its own T2 settlement re-applies too): -0.123410");
        assertEq(a.rh.eUsdg, 36_044_828_822, "RUN-hold: entity realises 36,044.828822");
        assertEq(a.ch.eUsdg, 13_526_811_094, "CONTROL-hold: entity realises 13,526.811094");
        assertEq(a.rh.eUsdg - a.ch.eUsdg, 22_518_017_728, "entity gain 22,518.017728");
        assertEq(a.ch.seniorPrincipal, 40_000e6, "CONTROL-hold = project expectation: senior 40,000");
        assertEq(a.ch.seniorPrincipal - a.rh.seniorPrincipal, 24_999_863_014, "Senior principal -24,999.863014");
        assertEq(a.ch.senLp - a.rh.senLp, 22_727_129_327, "SEN_LP -22,727.129327");
    }

    /// ENTITY PROFIT, the project's SC-L1 regression state. Without backing nothing is realisable (Junior principal
    /// stays 0, requests refunded) in every arm. If the liability ends after the entity's settlement: 44,535.797448 USDG
    /// at T2; the project's expected flow (= the fixed code): 0.
    function test_EntityProfit_ProjectState() public {
        Arms memory a = _allArms(false);
        assertEq(a.r2.eUsdg + a.c2.eUsdg, 0, "no backing: nothing realisable");
        assertEq(a.r2.hwm, 50_040e6, "RUN: Senior recovery target 100,000 -> 50,040 without backing");
        assertEq(a.ch.eUsdg, 0, "CONTROL-hold = project expectation: entity 0");
        assertEq(a.ch.seniorPrincipal, 100_000e6, "CONTROL-hold: senior 100,000");
        assertEq(a.rh.eUsdg, 44_535_797_448, "RUN-hold: entity realises 44,535.797448");
        assertEq(a.ch.seniorPrincipal - a.rh.seniorPrincipal, 49_960e6, "Senior principal -49,960");
        assertEq(a.ch.senLp - a.rh.senLp, 45_418_144_113, "SEN_LP -45,418.144113");
    }

    function _mixedFlow(
        bool extra
    ) internal returns (Book memory first, Book memory mid, Book memory fin) {
        _mixedState();
        first = _book();
        if (extra) {
            _ordinarySettlement();
        } else {
            _sameTimeNoCall();
        }
        mid = _book();
        _release(address(juniorVault));
        fin = _book();
    }

    /// IMMEDIATE FORM, the project's mixed test + one ordinary settlement (JUN_LP's full-balance request, ANYONE
    /// settles), no value arriving: Senior 25,000 -> 15,000, JUN_LP paid 9,090.999660 USDG in the same settlement.
    function test_MixedInflows_oneOrdinarySettlement_movesSeniorToJunior() public {
        uint256 snap = vm.snapshotState();
        (Book memory c1, Book memory c2, Book memory c) = _mixedFlow(false);
        vm.revertToState(snap);
        (Book memory r1, Book memory r2, Book memory r) = _mixedFlow(true);
        assertEq(keccak256(abi.encode(c1)), keccak256(abi.encode(r1)), "same state before the added call");
        _log("MIXED CONTROL, same moment (no call)", c2);
        _log("MIXED RUN after ONE ordinary settlement (no value arrived)", r2);
        assertEq(c2.seniorPrincipal, 25_000e6, "CONTROL: senior 25,000");
        assertEq(r2.seniorPrincipal, 15_000e6, "RUN: senior principal 25,000 -> 15,000");
        assertEq(r2.hwm, 15_000e6, "RUN: recovery target 40,000 -> 15,000");
        assertEq(r2.pendingRecap, 0, "RUN: residual recapitalization consumed without backing");
        assertEq(r2.junLpUsdg, 9_090_999_660, "RUN: JUN_LP paid 9,090.999660 USDG in the same settlement");
        assertEq(r2.juniorPrincipal + r2.junLpUsdg, 10_000e6, "RUN: 10,000 moved to Junior");
        assertEq(r2.poolRaw + r2.junLpUsdg, c2.poolRaw, "the USDG paid to JUN_LP left the pool");
        assertEq(c2.senLp - r2.senLp, 9_090_901_544, "SEN_LP -9,090.901544 at once");
        _log("MIXED CONTROL final (= project)", c);
        _log("MIXED RUN final", r);
        assertEq(c.seniorPrincipal, 40_000e6, "CONTROL = project: senior 40,000");
        assertEq(c.juniorPrincipal, 15_000e6, "CONTROL = project: junior 15,000");
        assertEq(r.seniorPrincipal, 15_000e6, "RUN: senior 15,000");
        assertEq(r.juniorPrincipal + r.junLpUsdg, 40_000e6, "RUN: junior 40,000 incl. what JUN_LP took out");
        assertEq(r.poolRaw + r.junLpUsdg, c.poolRaw, "conserved");
    }

    /// The project's own assertions, as external calls so a failure can be caught.
    function projectAssertionsAfterFirstReconcile() external view {
        assertEq(pool.seniorPrincipal(), SETTLEABLE, "Live reconcile should apply only the settleable recap");
        assertEq(pool.seniorHighWaterMark(), PENDING, "Residual recap should preserve senior's recovery target");
        assertEq(pool.pendingRecapitalizationAssets(), CLAIM, "Unsettled recap must remain pending");
    }

    function projectAssertionsAfterRelease() external view {
        assertEq(pool.seniorPrincipal(), PENDING, "Residual recap should settle once backing becomes available");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "Fully settled recap should clear the pending bucket");
        assertEq(pool.unassignedAssets(), 0, "Settled recap should keep senior restoration out of unassigned assets");
    }

    function _failureOf(
        bytes4 selector
    ) internal view returns (string memory reason) {
        (bool ok, bytes memory ret) = address(this).staticcall(abi.encodeWithSelector(selector));
        if (ok) {
            return "";
        }
        assembly {
            ret := add(ret, 4)
        }
        reason = abi.decode(ret, (string));
    }

    /// The project's SC-L1 regression test + one ordinary settlement: its own assertions fail on the observed state.
    function test_ProjectTest_withOneOrdinarySettlement() public {
        _projectState();
        assertEq(_ordinarySettlement().juniorProcessedEpochs, 1, "JUN_LP's epoch processed (refund at zero NAV)");
        Book memory mid = _book();
        _log("after ONE ordinary settlement (no value arrived, trader claim unchanged)", mid);
        assertEq(mid.poolRaw, PENDING, "no USDG entered or left the pool");
        assertEq(engine.totalTraderClaimBalanceAssets(), CLAIM, "trader claim unchanged");
        assertEq(mid.seniorPrincipal, SETTLEABLE, "Senior principal unchanged (49,960)");
        assertEq(mid.hwm, CLAIM, "recovery target 100,000 -> 50,040");
        assertEq(mid.pendingRecap, CLAIM - SETTLEABLE, "pending recap 50,040 -> 80");
        assertEq(
            _failureOf(this.projectAssertionsAfterFirstReconcile.selector),
            "Residual recap should preserve senior's recovery target: 50040000000 != 100000000000",
            "the project's own assertion fails"
        );
        _release(address(seniorVault));
        Book memory fin = _book();
        _log("after the project's release + reconcile", fin);
        assertEq(fin.seniorPrincipal, CLAIM, "Senior 50,040 (project: 100,000)");
        assertEq(fin.juniorPrincipal, 49_880e6, "Junior 49,880 (project: 0)");
        assertEq(fin.unassigned, 80e6, "unassigned 80 (project: 0)");
        assertEq(
            _failureOf(this.projectAssertionsAfterRelease.selector),
            "Residual recap should settle once backing becomes available: 50040000000 != 100000000000",
            "the project's own final assertion fails"
        );
    }

    /// CONTROL: the same test and clock without the added calls; the project's assertions hold.
    function test_Control_noSettlement_projectAssertionsHold() public {
        _projectState();
        _sameTimeNoCall();
        assertEq(bytes(_failureOf(this.projectAssertionsAfterFirstReconcile.selector)).length, 0, "project ok");
        _release(address(seniorVault));
        assertEq(bytes(_failureOf(this.projectAssertionsAfterRelease.selector)).length, 0, "project final ok");
        assertEq(pool.juniorPrincipal(), 0, "Junior 0: recapitalization never accrues to junior principal");
    }

    /// RUN minus CONTROL per party, same snapshot and clock: value moves from Senior holders to Junior holders.
    function test_RunMinusControl_perParty() public {
        _projectState();
        uint256 snap = vm.snapshotState();
        _sameTimeNoCall();
        _release(address(seniorVault));
        Book memory c = _book();
        uint256 tC = block.timestamp;
        vm.revertToState(snap);
        _ordinarySettlement();
        assertEq(block.timestamp, tC, "same clock in both arms");
        _release(address(seniorVault));
        Book memory r = _book();
        _delta("Senior principal RUN - CONTROL", r.seniorPrincipal, c.seniorPrincipal);
        _delta("Junior principal RUN - CONTROL", r.juniorPrincipal, c.juniorPrincipal);
        _delta("SEN_LP RUN - CONTROL          ", r.senLp, c.senLp);
        _delta("Senior seed RUN - CONTROL     ", r.senSeed, c.senSeed);
        _delta("JUN_LP RUN - CONTROL          ", r.junLp, c.junLp);
        _delta("Junior seed RUN - CONTROL     ", r.junSeed, c.junSeed);
        assertEq(r.poolRaw, c.poolRaw, "same pool cash in both arms");
        assertEq(
            r.seniorPrincipal + r.juniorPrincipal + r.unassigned + r.pendingRecap,
            c.seniorPrincipal + c.juniorPrincipal + c.unassigned + c.pendingRecap,
            "value conserved"
        );
        assertEq(c.seniorPrincipal - r.seniorPrincipal, 49_960e6, "Senior holders -49,960");
        assertEq(r.juniorPrincipal - c.juniorPrincipal, 49_880e6, "Junior holders +49,880");
        assertEq(r.unassigned - c.unassigned, 80e6, "+80 unassigned");
        assertEq(c.senLp - r.senLp, 45_418_144_113, "SEN_LP -45,418.144113");
        assertEq(r.junLp - c.junLp, 45_345_906_306, "JUN_LP +45,345.906306");
    }

    /// @dev Exact-gap form: the owner's recapitalization equals the 50,000 deficit; the backing then arrives in ten
    ///      steps, modelled as in the project's test (trader-claim liability falls). RUN: one ordinary settlement after
    ///      steps 1-4; CONTROL: none. Final values from the project's preview getPendingTrancheState().
    function _exactGap(
        bool run
    ) internal returns (uint256 sFinal, uint256 jFinal, uint256 hwm4) {
        _wipe();
        _setTotalTraderClaim(50_000e6);
        _ownerRecapitalizes(50_000e6);
        vm.warp(block.timestamp + 1 hours + 1);
        for (uint256 i = 1; i <= 10; ++i) {
            _setTotalTraderClaim(50_000e6 - i * 5_000e6);
            if (run && i <= 4) {
                _ordinarySettlement();
                console.log("RUN step / Senior HWM", i, pool.seniorHighWaterMark());
            } else {
                _sameTimeNoCall();
            }
            if (i == 4) {
                hwm4 = pool.seniorHighWaterMark();
            }
            vm.warp(block.timestamp + 1 hours + 1);
        }
        (sFinal, jFinal,,) = pool.getPendingTrancheState();
    }

    /// Routine operation: settlements while the backing arrives step by step.
    function test_ExactGapRecap_routineSettlements_vs_none() public {
        uint256 snap = vm.snapshotState();
        (uint256 cS, uint256 cJ,) = _exactGap(false);
        uint256 tC = block.timestamp;
        _ordinarySettlement(); // the preview is what a settlement does
        assertEq(pool.seniorPrincipal(), cS, "CONTROL: preview equals the settled Senior principal");
        assertEq(pool.juniorPrincipal(), cJ, "CONTROL: preview equals the settled Junior principal");
        vm.revertToState(snap);
        (uint256 rS, uint256 rJ, uint256 rHwm4) = _exactGap(true);
        assertEq(block.timestamp, tC, "same clock");
        console.log("CONTROL / RUN final Senior", cS, rS);
        console.log("CONTROL / RUN final Junior", cJ, rJ);
        assertEq(cS, 50_000e6, "CONTROL: the recapitalization restores Senior");
        assertEq(cJ, 0, "CONTROL: nothing to Junior");
        assertEq(rHwm4, 20_000e6, "RUN: recovery target 50,000 -> 20,000 in four settlements");
        assertEq(rS, 20_000e6, "RUN: Senior keeps 20,000");
        assertEq(rJ, 30_000e6, "RUN: 30,000 of the owner's recapitalization ends as Junior principal");
    }

    /// Supporting facts for the entity: no Junior entry by deposit after the wipe (refused before and after the
    /// recapitalization), and with nothing queued a settlement reverts, so E_SMALL's request is what enables it.
    function test_NoDepositEntry_and_settlementNeedsAQueuedRequest() public {
        settlementToken.mint(E_MAIN, 10_000e6);
        vm.prank(E_MAIN);
        settlementToken.approve(address(juniorVault), type(uint256).max);
        uint256 snap = vm.snapshotState();
        _wipe();
        vm.prank(E_MAIN);
        vm.expectRevert(TrancheVault.TrancheVault__TerminallyWiped.selector);
        juniorVault.requestDeposit(10_000e6, E_MAIN, E_MAIN);
        vm.revertToState(snap);
        _mixedState();
        vm.prank(E_MAIN);
        (bool ok, bytes memory ret) = address(juniorVault).call(
            abi.encodeWithSignature("requestDeposit(uint256,address,address)", 10_000e6, E_MAIN, E_MAIN)
        );
        assertFalse(ok, "Junior requestDeposit after the recapitalization and first reconcile reverts");
        console.logBytes4(bytes4(ret));
        vm.warp((block.timestamp / 1 hours + 1) * 1 hours);
        vm.prank(E_MAIN);
        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        pool.settleLpEpoch(0, 0);
    }

}

/// Whole state from transactions. Accounts are funded once (USDG to wallets at setUp); LPs deposit through
/// requestDeposit -> permissionless settleLpEpoch -> claimDeposit; traders trade through OrderRouter with a keeper
/// (MockPyth prices); the mark moves to the 2.00 cap and a SHORT closes there, so pool cash net of claims falls below the
/// liability; the owner calls only CfdEngine.recapitalizePool and CfdEngine.clearDegradedMode (runbook "4. Recapitalize
/// ... 5. Clear degraded mode only after raw solvency E >= L"). No vm.store / stdstore, no prank as a protocol contract,
/// no mint into a protocol contract. Owner configuration: the project's setUp and one Senior-capacity pool config
/// before any deposit. engine.degradedMode() is asserted after every step; every refused call by its selector.
/// RUN = keeper settlements at the cap until n succeed; CONTROL = exactly one, same epochs, same holder calls; then the
/// same unwind (price back to 1.00, SHORTs closed, every holder redeems) and final amounts per party RUN - CONTROL.
contract ResidualRecapFromTransactionsTest is BasePerpTest {

    address internal constant PAIR_LONG = address(0xA1);
    address internal constant PAIR_SHORT = address(0xA2);
    address internal constant NET_TRADER = address(0xA3);
    address internal constant NET_TRADER2 = address(0xA4);
    address internal constant KEEPER = address(0xBEEF);
    address internal constant JUN_A = address(0xC1); // Junior 700,000
    address internal constant JUN_B = address(0xC2); // Junior 300,000, requests its full balance (the queue item)
    address internal constant SEN_A = address(0xC4); // Senior 50,000

    uint256 internal constant LOT = 1e20;
    uint256 internal constant LOT_ASSETS_AT_PAR = 100e6;
    uint256 internal constant FAR = 2e8; // the 2.00 cap
    bytes4 internal constant SEL_DEGRADED = IHousePool.HousePool__DegradedMode.selector;
    bytes4 internal constant SEL_NO_PROGRESS = IHousePool.HousePool__NoLpEpochProgress.selector;
    bytes4 internal constant SEL_NONE = bytes4(0xffffffff); // no call: holder has no shares
    bytes4 internal constant SEL_COOLDOWN = TrancheVault.TrancheVault__DepositCooldown.selector;
    uint256 internal constant NP = 16;

    uint256 internal px = 1e8;
    uint256 internal pairShortLots;
    mapping(address => uint256[]) internal jIds;
    mapping(address => uint256[]) internal sIds;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams(0.01e18, 0.4e18, 10, 20, 300, 500, 1e6, 10, 5000, 0);
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        settlementToken.mint(JUN_A, 700_000e6);
        settlementToken.mint(JUN_B, 300_000e6);
        settlementToken.mint(SEN_A, 50_000e6);
        settlementToken.mint(NET_TRADER, 60_000e6);
        settlementToken.mint(NET_TRADER2, 60_000e6);
        settlementToken.mint(PAIR_LONG, 300_000e6);
        settlementToken.mint(PAIR_SHORT, 300_000e6 + 1e6);
        settlementToken.mint(address(this), 5_000_000e6); // the owner's wallet
        _dm(false, "setUp");
        _deposit(juniorVault, JUN_A, 700_000e6);
        _deposit(juniorVault, JUN_B, 300_000e6);
        _deposit(seniorVault, SEN_A, 50_000e6);
    }

    function _dm(
        bool expected,
        string memory step
    ) internal view {
        bool d = engine.degradedMode();
        console.log(string.concat("   [degradedMode=", d ? "true" : "false", "] ", step));
        assertEq(d, expected, string.concat("degradedMode at step: ", step));
    }

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _step(
        uint256 secs
    ) internal {
        vm.warp(_now() + secs);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _toLive() internal {
        for (uint256 i; i < 60 && (engine.isOracleFrozen() || engine.isFadWindow()); ++i) {
            vm.warp(_now() + 3 hours);
        }
    }

    function _liveData(
        uint256 p
    ) internal view returns (bytes[] memory d) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            u[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(p)), 0, -8, _now(), engine.lastMarkTime()));
        }
        d = new bytes[](1);
        d[0] = abi.encode(u);
    }

    function _pushMark(
        uint256 p
    ) internal {
        px = p;
        bytes[] memory d = _liveData(p);
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    function _deposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 id = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        uint256 t = vault.depositEpochStart(id);
        if (_now() < t) {
            vm.warp(t);
        }
        _toLive();
        _step(1);
        _pushMark(px);
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        uint256 c = vault.claimableDepositRequest(id, lp);
        assertEq(c, amount, "deposit finalized");
        vm.prank(lp);
        vault.claimDeposit(id, c, lp, lp);
        _dm(false, "LP deposit");
    }

    function _depositTrader(
        address trader,
        uint256 amount
    ) internal {
        vm.startPrank(trader);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.deposit(trader, amount);
        vm.stopPrank();
    }

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin,
        bool isClose,
        uint256 p
    ) internal {
        _step(1);
        _pushMark(p);
        vm.prank(account);
        uint64 id = router.commitOrder(side, lots * LOT, margin, 0, isClose);
        bytes[] memory upd = _mockPythUpdateData(p);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, upd);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    /// @dev Keeper settlement: OrderRouter path with open positions, HousePool path without (both permissionless).
    function _settle() internal returns (bytes4) {
        bytes[] memory d = _liveData(px);
        if (engine.terminalNavSnapshot().hasOpenPositions) {
            vm.prank(KEEPER);
            try router.settleLpEpoch(d) {
                return bytes4(0);
            } catch (bytes memory err) {
                return bytes4(err);
            }
        }
        vm.prank(KEEPER);
        try pool.settleLpEpoch(0, 0) {
            return bytes4(0);
        } catch (bytes memory err2) {
            return bytes4(err2);
        }
    }

    function _epoch() internal {
        _step(1 hours + 1);
        _toLive();
        _pushMark(px);
    }

    function _effective() internal view returns (uint256) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return p > c ? p - c : 0;
    }

    function _logPool(
        string memory tag
    ) internal view {
        console.log(tag);
        console.log("   S / J stored / Senior HWM", pool.seniorPrincipal(), pool.juniorPrincipal(), pool.seniorHighWaterMark());
        console.log("   pending revenue / recap  ", pool.pendingTradingRevenueAssets(), pool.pendingRecapitalizationAssets());
        console.log("   E / L                    ", _effective(), _maxLiability());
    }

    function _req(
        TrancheVault vault,
        mapping(address => uint256[]) storage ids,
        address lp,
        string memory who
    ) internal {
        uint256 shares = vault.balanceOf(lp);
        if (shares == 0) {
            return;
        }
        vm.prank(lp);
        try vault.requestRedeem(shares, lp, lp) returns (uint256 id) {
            ids[lp].push(id);
        } catch (bytes memory err) {
            // refunded shares restart the deposit cooldown: the only accepted refusal
            assertEq(bytes4(err), SEL_COOLDOWN, string.concat("holder request refused, not DepositCooldown: ", who));
        }
    }

    function _coll(
        TrancheVault vault,
        uint256[] storage ids,
        address lp
    ) internal {
        for (uint256 i; i < ids.length; ++i) {
            if (vault.refundableRedeemRequest(ids[i], lp) > 0) {
                vm.prank(lp);
                vault.claimRedeemRefund(ids[i], lp, lp);
            }
            uint256 c = vault.claimableRedeemRequest(ids[i], lp);
            if (c > 0) {
                vm.prank(lp);
                vault.claimRedeem(ids[i], c, lp, lp);
            }
        }
    }

    /// @dev Two net SHORTs and a near-balanced pair book; the pair LONG closes at 0.985, the mark moves to the 2.00 cap
    ///      and NET_TRADER's SHORT closes there.
    function _wipeByTrades() internal {
        _depositTrader(NET_TRADER, 60_000e6);
        _depositTrader(NET_TRADER2, 60_000e6);
        _trade(NET_TRADER, CfdTypes.Side.SHORT, 2_500, 20_000e6, false, 1e8);
        _trade(NET_TRADER2, CfdTypes.Side.SHORT, 1_000, 20_000e6, false, 1e8);
        _dm(false, "two net SHORTs open");
        uint256 e0 = _effective();
        uint256 pairLots = (e0 * 10_000) / (10_025 * LOT_ASSETS_AT_PAR) - 10;
        uint256 maxSkewLots = (e0 * 4) / (10 * LOT_ASSETS_AT_PAR) - 1;
        _depositTrader(PAIR_LONG, 300_000e6);
        _depositTrader(PAIR_SHORT, 300_000e6);
        uint256 shortLots = pairLots - 3_500;
        pairShortLots = shortLots;
        uint256 l1 = 3_500 + maxSkewLots - 50;
        _trade(PAIR_LONG, CfdTypes.Side.LONG, l1, 40_000e6, false, 1e8);
        uint256 s1 = l1 - 3_500 + maxSkewLots - 50 < shortLots ? l1 - 3_500 + maxSkewLots - 50 : shortLots;
        _trade(PAIR_SHORT, CfdTypes.Side.SHORT, s1, 40_000e6, false, 1e8);
        if (pairLots > l1) {
            _trade(PAIR_LONG, CfdTypes.Side.LONG, pairLots - l1, 10_000e6, false, 1e8);
        }
        if (shortLots > s1) {
            _trade(PAIR_SHORT, CfdTypes.Side.SHORT, shortLots - s1, 10_000e6, false, 1e8);
        }
        _dm(false, "pair book open");
        _step(60);
        _trade(PAIR_LONG, CfdTypes.Side.LONG, pairLots, 0, true, 0.985e8);
        _dm(true, "PAIR_LONG closes at 0.985 (pool cash net of claims below max liability)");
        _step(60);
        _pushMark(FAR);
        _dm(true, "keeper pushes the 2.00 cap mark");
        _trade(NET_TRADER, CfdTypes.Side.SHORT, 2_500, 0, true, FAR);
        _dm(true, "NET_TRADER closes SHORT at the cap (closes allowed in degraded mode)");
        _logPool("after the wipe");
    }

    /// @dev The owner's documented routine with every refused step asserted; `partialFirst` first funds half the gap.
    function _ownerProcedure(
        bool partialFirst
    ) internal {
        _epoch();
        _req(juniorVault, jIds, JUN_B, "JUN_B");
        _dm(true, "JUN_B requests redemption of its full balance (accepted)");
        _epoch();
        assertEq(_settle(), SEL_DEGRADED, "settlement before the owner acts: HousePool__DegradedMode");
        _dm(true, "keeper settleLpEpoch refused");
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();
        _dm(true, "owner clearDegradedMode before recapitalization refused (StillInsolvent)");
        uint256 gap = _maxLiability() - _effective();
        console.log("gap L - E", gap);
        settlementToken.approve(address(engine), gap);
        if (partialFirst) {
            engine.recapitalizePool(gap / 2);
            _dm(true, "owner recapitalizePool(half of the gap)");
            assertEq(_settle(), SEL_DEGRADED, "settlement after a partial recapitalization: HousePool__DegradedMode");
            vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
            engine.clearDegradedMode();
            _dm(true, "settlement and clearDegradedMode after the partial recapitalization refused");
            gap = _maxLiability() - _effective();
        }
        engine.recapitalizePool(gap);
        _dm(true, "owner recapitalizePool(gap); degraded mode not cleared automatically");
        assertEq(_settle(), SEL_DEGRADED, "settlement after recapitalization, before clear: HousePool__DegradedMode");
        _dm(true, "keeper settleLpEpoch before clear refused");
        engine.clearDegradedMode();
        _dm(false, "owner clearDegradedMode (E >= L)");
    }

    /// @dev Common start: wipe, owner routine, the first ordinary settlement (normal reconciliation), a day, then a
    ///      trader's margin deposit collects carry and queues a Revenue bucket.
    function _start(
        bool partialFirst
    ) internal returns (uint256 bucket) {
        _wipeByTrades();
        _ownerProcedure(partialFirst);
        assertEq(_settle(), bytes4(0), "first ordinary settlement after clearDegradedMode succeeds");
        _dm(false, "first keeper settlement (normal reconciliation)");
        _logPool("after the first ordinary settlement");
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 0, "both stored principals zero");
        assertGt(seniorVault.totalSupply(), 0, "Senior shares outstanding");
        assertEq(pool.pendingRecapitalizationAssets(), 7_243_528_343, "the whole recapitalization still pending");
        _coll(juniorVault, jIds[JUN_B], JUN_B);
        _req(juniorVault, jIds, JUN_B, "JUN_B");
        _step(1 days);
        _epoch();
        _dm(false, "JUN_B re-requests; one day passes");
        uint256 rev0 = pool.pendingTradingRevenueAssets();
        vm.startPrank(PAIR_SHORT);
        settlementToken.approve(address(clearinghouse), 1e6);
        clearinghouse.deposit(PAIR_SHORT, 1e6);
        vm.stopPrank();
        bucket = pool.pendingTradingRevenueAssets() - rev0;
        _dm(false, "PAIR_SHORT deposits 1 USDG margin (carry -> Revenue bucket)");
        assertEq(bucket, 94_347_204, "Revenue bucket 94.347204");
    }

    struct Fin {
        uint256 okSettles;
        uint256 epochs;
        uint256 hwmAtCap;
        uint256 recapAtCap;
        uint256[NP] v;
        uint256 partySum;
        uint256 tokenSupply;
        uint256 tokenKnown;
    }

    function _names() internal pure returns (string[NP] memory n) {
        n[0] = "SEN_A (Senior LP)";
        n[1] = "Senior seed";
        n[2] = "JUN_A (Junior LP)";
        n[3] = "JUN_B (Junior LP, trigger)";
        n[4] = "Junior seed";
        n[5] = "PAIR_SHORT";
        n[6] = "NET_TRADER2";
        n[7] = "NET_TRADER";
        n[8] = "PAIR_LONG";
        n[9] = "KEEPER";
        n[10] = "treasury";
        n[11] = "pool unassignedAssets";
        n[12] = "pending recap (not summed)";
        n[13] = "pending revenue (not summed)";
        n[14] = "pool remainder (not summed)";
        n[15] = "owner wallet";
    }

    function _capPhase(
        uint256 target,
        uint256 epochs,
        Fin memory f
    ) internal {
        uint256 sinceOk;
        for (uint256 i; i < 400; ++i) {
            if (epochs == 0 && (f.okSettles >= target || sinceOk >= 40)) {
                break;
            }
            if (epochs > 0 && i >= epochs) {
                break;
            }
            _epoch();
            f.epochs += 1;
            sinceOk += 1;
            if (f.okSettles < target) {
                bytes4 sel = _settle();
                if (sel == bytes4(0)) {
                    sinceOk = 0;
                    f.okSettles += 1;
                } else {
                    assertEq(sel, SEL_NO_PROGRESS, "refused keeper settlement at the cap: NoLpEpochProgress only");
                }
            }
            _coll(juniorVault, jIds[JUN_B], JUN_B);
            _req(juniorVault, jIds, JUN_B, "JUN_B");
            assertFalse(engine.degradedMode(), "degradedMode false in every cap-phase epoch");
        }
        _dm(false, "end of the cap phase");
        f.hwmAtCap = pool.seniorHighWaterMark();
        f.recapAtCap = pool.pendingRecapitalizationAssets();
    }

    function _unwind(
        Fin memory
    ) internal {
        _step(60);
        _pushMark(1e8);
        _trade(PAIR_SHORT, CfdTypes.Side.SHORT, pairShortLots, 0, true, 1e8);
        _trade(NET_TRADER2, CfdTypes.Side.SHORT, 1_000, 0, true, 1e8);
        _dm(false, "price back to 1.00, both SHORTs closed");
        for (uint256 k; k < 6; ++k) {
            _coll(juniorVault, jIds[JUN_A], JUN_A);
            _coll(juniorVault, jIds[JUN_B], JUN_B);
            _coll(seniorVault, sIds[SEN_A], SEN_A);
            _step(1 hours + 1);
            _toLive();
            _pushMark(1e8);
            _req(juniorVault, jIds, JUN_A, "JUN_A");
            _req(juniorVault, jIds, JUN_B, "JUN_B");
            _req(seniorVault, sIds, SEN_A, "SEN_A");
            _epoch();
            bytes4 us = _settle();
            assertTrue(us == bytes4(0) || us == SEL_NO_PROGRESS, "unwind settlement: ok or NoLpEpochProgress");
            assertFalse(engine.degradedMode(), "degradedMode false in every unwind round");
        }
        _coll(juniorVault, jIds[JUN_A], JUN_A);
        _coll(juniorVault, jIds[JUN_B], JUN_B);
        _coll(seniorVault, sIds[SEN_A], SEN_A);
        _dm(false, "end of the unwind");
    }

    function _escrow(
        TrancheVault vault,
        uint256[] storage ids,
        address lp
    ) internal view returns (uint256 v) {
        uint256 pend;
        for (uint256 i; i < ids.length; ++i) {
            pend += vault.pendingRedeemRequest(ids[i], lp) + vault.refundableRedeemRequest(ids[i], lp);
            v += vault.claimableRedeemAssets(ids[i], lp);
        }
        v += vault.convertToAssets(pend);
    }

    function _lp(
        TrancheVault vault,
        uint256[] storage ids,
        address lp
    ) internal view returns (uint256) {
        return settlementToken.balanceOf(lp) + vault.convertToAssets(vault.balanceOf(lp)) + _escrow(vault, ids, lp);
    }

    function _trader(
        address a
    ) internal view returns (uint256) {
        return settlementToken.balanceOf(a) + clearinghouse.balanceAssets(a) + engine.traderClaimBalanceAssets(a);
    }

    function _measure(
        Fin memory f
    ) internal view {
        f.v[0] = _lp(seniorVault, sIds[SEN_A], SEN_A);
        f.v[1] = seniorVault.convertToAssets(seniorVault.balanceOf(address(this)));
        f.v[2] = _lp(juniorVault, jIds[JUN_A], JUN_A);
        f.v[3] = _lp(juniorVault, jIds[JUN_B], JUN_B);
        f.v[4] = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        f.v[5] = _trader(PAIR_SHORT);
        f.v[6] = _trader(NET_TRADER2);
        f.v[7] = _trader(NET_TRADER);
        f.v[8] = _trader(PAIR_LONG);
        f.v[9] = _trader(KEEPER);
        f.v[10] = _trader(engine.protocolTreasury());
        f.v[11] = pool.unassignedAssets();
        f.v[12] = pool.pendingRecapitalizationAssets();
        f.v[13] = pool.pendingTradingRevenueAssets();
        uint256 owned = pool.seniorPrincipal() + pool.juniorPrincipal() + f.v[11] + f.v[12] + f.v[13];
        uint256 raw = pool.rawAssets();
        f.v[14] = raw > owned ? raw - owned : 0;
        f.v[15] = settlementToken.balanceOf(address(this));
        for (uint256 i; i < NP; ++i) {
            if (i < 12 || i == 15) {
                f.partySum += f.v[i];
            }
        }
        f.tokenSupply = settlementToken.totalSupply();
        address[15] memory who = [
            SEN_A, JUN_A, JUN_B, PAIR_SHORT, NET_TRADER2, NET_TRADER, PAIR_LONG, KEEPER, engine.protocolTreasury(),
            address(this), address(pool), address(clearinghouse), address(juniorVault), address(seniorVault),
            address(engine)
        ];
        for (uint256 i; i < who.length; ++i) {
            f.tokenKnown += settlementToken.balanceOf(who[i]);
        }
    }

    function _arm(
        uint256 target,
        uint256 epochs
    ) internal returns (Fin memory f) {
        _capPhase(target, epochs, f);
        _unwind(f);
        _measure(f);
    }

    function _signed(
        uint256 r,
        uint256 c
    ) internal pure returns (string memory) {
        return r >= c ? string.concat("+", vm.toString(r - c)) : string.concat("-", vm.toString(c - r));
    }

    function _runVsControl(
        uint256 n
    ) internal returns (Fin memory r, Fin memory c, uint256 bucket) {
        bucket = _start(false);
        uint256 snap = vm.snapshotState();
        r = _arm(n, 0);
        vm.revertToState(snap);
        c = _arm(1, r.epochs);
        assertEq(c.okSettles, 1, "CONTROL: exactly one settlement at the cap");
        assertEq(r.okSettles, n, "RUN: n settlements at the cap");
        assertEq(r.epochs, c.epochs, "same epochs in both arms");
        console.log("cap: Senior HWM RUN / CONTROL", r.hwmAtCap, c.hwmAtCap);
        console.log("cap: pending recap RUN / CONTROL", r.recapAtCap, c.recapAtCap);
        string[NP] memory nm = _names();
        console.log("final amounts: party | RUN | CONTROL | RUN - CONTROL");
        for (uint256 i; i < NP; ++i) {
            console.log(string.concat("   ", nm[i], " | ", vm.toString(r.v[i]), " | ", vm.toString(c.v[i]), " | ", _signed(r.v[i], c.v[i])));
        }
        console.log(string.concat("Senior holders RUN - CONTROL ", _signed(r.v[0] + r.v[1], c.v[0] + c.v[1])));
        console.log(string.concat("Junior holders RUN - CONTROL ", _signed(r.v[2] + r.v[3] + r.v[4], c.v[2] + c.v[3] + c.v[4])));
        assertEq(r.tokenSupply, c.tokenSupply, "same USDG supply in both arms");
        assertEq(r.tokenKnown, r.tokenSupply, "RUN: every USDG at a listed address");
        assertEq(c.tokenKnown, c.tokenSupply, "CONTROL: every USDG at a listed address");
        assertApproxEqAbs(r.partySum, c.partySum, 2, "conservation: sum of all final amounts RUN == CONTROL");
        for (uint256 i = 5; i <= 10; ++i) {
            assertEq(r.v[i], c.v[i], "traders, keeper, treasury: unchanged");
        }
        // Each extra settlement at the cap applies the Revenue bucket again as recapitalization: pending recap and the
        // Senior HWM fall by one bucket per extra settlement (the project expects them to stay).
        assertEq(c.hwmAtCap - r.hwmAtCap, (n - 1) * bucket, "Senior HWM falls one bucket per extra settlement");
        assertEq(c.recapAtCap - r.recapAtCap, (n - 1) * bucket, "pending recap falls one bucket per extra settlement");
        assertEq(c.v[11] - r.v[11], (n - 1) * bucket, "unassigned (owner's recapitalization) -1 bucket per extra");
    }

    /// Owner procedure gates: before clearDegradedMode every settlement reverts HousePool__DegradedMode (also after a
    /// partial and after the full recapitalization); the clear reverts StillInsolvent until the gap is paid. After the
    /// exact gap and the clear the first settlement succeeds and the whole recapitalization is still pending.
    function test_FromTransactions_ownerProcedureGates() public {
        _start(true);
    }

    /// One extra keeper settlement (n = 2): Senior holders -94.348184, Junior holders +188.695387.
    function test_FromTransactions_oneExtraSettlement() public {
        (Fin memory r, Fin memory c,) = _runVsControl(2);
        assertEq(c.v[0] - r.v[0], 92_496_017, "SEN_A -92.496017");
        assertEq((c.v[0] + c.v[1]) - (r.v[0] + r.v[1]), 94_348_184, "Senior holders -94.348184");
        assertEq((r.v[2] + r.v[3] + r.v[4]) - (c.v[2] + c.v[3] + c.v[4]), 188_695_387, "Junior holders +188.695387");
    }

    /// 59 extra keeper settlements over 120 epochs (n = 60): SEN_A 7,101.403002 -> 1,644.138024.
    function test_FromTransactions_59ExtraSettlements() public {
        (Fin memory r, Fin memory c,) = _runVsControl(60);
        assertEq(c.v[0], 7_101_403_002, "CONTROL: SEN_A ends with 7,101.403002");
        assertEq(r.v[0], 1_644_138_024, "RUN: SEN_A ends with 1,644.138024");
        assertEq((c.v[0] + c.v[1]) - (r.v[0] + r.v[1]), 5_566_545_883, "Senior holders -5,566.545883");
        assertEq((r.v[2] + r.v[3] + r.v[4]) - (c.v[2] + c.v[3] + c.v[4]), 11_133_030_918, "Junior holders +11,133.030918");
        assertEq(r.v[3] - c.v[3], 3_336_556_720, "JUN_B (trigger) +3,336.556720");
    }

}
```

