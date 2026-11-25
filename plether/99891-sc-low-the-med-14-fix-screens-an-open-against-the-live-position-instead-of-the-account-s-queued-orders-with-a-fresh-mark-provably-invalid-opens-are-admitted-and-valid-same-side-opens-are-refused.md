# \#99891 [SC-Low] The MED-14 fix screens an open against the live position instead of the account's queued orders: with a fresh mark, provably invalid opens are admitted and valid same-side opens are refused

**Submitted on Nov 15th 2026 at 16:43:28 UTC by @argo_the_hacker for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99891
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

**Severity: Low.** Impact selected from the program's list: "Contract fails to deliver promised returns, but doesn't lose value".

**Target:** `immunefi-team/audit-comp-plether` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src/router/OrderLiquidationBatchLogic.sol`, `_validatedOpenBounty` (lines 404-455), mainly lines 423-437.

**Relation to prior findings:** a new bug introduced by, and an incomplete part of, the fix for Cecuro **MED-14** (commit `88ef3954`, #126, "fix(perps): validate open side against queued position"; Cecuro's MED-14 and MED-16 comments link plether-core `5029cde9` for this change). The program's known-issues rule makes a report on a fixed root valid when it shows the fix "is incomplete or introduces a new bug". This report shows both. It is **not** the open MED-16/LOW-57 item, which the Scope tab limits to "opens committed against a stale cached price": all four symptoms below need a **fresh** mark. The section "Relation to prior findings" explains each item.

## Summary

To fix MED-14, `_validatedOpenBounty` now projects the trader's live position through their earlier queued orders (`_queuedPosition`, `OrderLiquidationBatchLogic.sol:367-402`). It uses that projection for one thing only: the side check at `OrderLiquidationBatchLogic.sol:426-428`. The price-dependent screen that follows still calls `previewOpenRevertCode(account, …)` (`OrderLiquidationBatchLogic.sol:431-432`), and that call plans the new open on top of the **live** engine position, ignoring every order the account has already queued. Behind the account's own queued orders this gives the wrong answer in both directions, in four ways:

1. **Opposing open: provably invalid orders are admitted (new bug).** The live position is on the other side, so the planner returns `MUST_CLOSE_OPPOSING` (`CfdEnginePlanLib.sol:313-316`) before it evaluates skew, solvency or margin. The fix then exempts exactly that code from the fresh-mark screen (`OrderLiquidationBatchLogic.sol:437`, `if (commitMarkTime != 0 && revertCode != opposingCode)`; before `88ef3954` it read `if (commitMarkTime != 0)`). So **no** price-dependent check runs, even though the mark is fresh. In the PoC a trader with a queued close commits a $1M SHORT with zero margin, more than twice the maximum skew. It is admitted, and it fails at execution. The same order from a flat account is refused at commit with `PredictableOpenInvalid(SKEW_TOO_HIGH)`. The maintainers' MED-16 comment says "Price-dependent checks remain freshness-gated", that is, they are enforced while the mark is fresh. On this path they are not.
2. **Same-side open: valid orders are still refused (fix incomplete).** The live position is on the same side, so the planner sizes live + new, a position that never exists. In the PoC a trader who is LONG 300k and has queued a full close cannot queue LONG 200k: it is refused `PredictableOpenInvalid(SKEW_TOO_HIGH)`. The identical open is admitted and fills once the close has executed. This is MED-14's own impact ("Users cannot pre-queue the next open behind a closing order"), and it breaks `ACCOUNTING_SPEC.md:1096`: "An open must match the position projected through earlier queued orders."
3. **Same-side open: invalid orders are admitted (fix incomplete).** The same live + new sizing also works the other way: the live position's margin carries an open whose own margin is far too small. In the PoC a trader holds a small LONG ($10k), the price drops to 0.972, and they queue a full close. A LONG $10k with 14 USDC margin then reads `OK` for them and is admitted. The same order from a flat account is refused at commit with `PredictableOpenInvalid(INSUFFICIENT_INITIAL_MARGIN)` (the fixture's 150 bps initial margin on ~$9.7k is ~146 USDC; 160 USDC is admitted). After the close executes, the admitted open fails at execution.
4. **Same-side open behind a queued same-side open: valid orders are refused (fix incomplete).** The preview also ignores queued **opens**. In the PoC a flat trader queues SHORT $7.1k with 21% margin, then commits SHORT $23.8k with 1% margin. Planned on the live (flat) position, the second order is refused `PredictableOpenInvalid(INSUFFICIENT_INITIAL_MARGIN)`. On the projected $30.9k position it carries ~1,729 USDC against an initial margin of ~464 USDC, and once the first open has executed the identical request is admitted and fills.

One fix cures all four: run the price-dependent screen against the position projected through the account's queued orders, not the live position. That is Cecuro's own MED-14 remediation text. **Restoring line 437 to `if (commitMarkTime != 0)`, the evident fix for symptom 1, does not cure symptom 3, and it undoes the MED-14 fix.** In symptom 3 the screen already runs (the code is `OK`, not `MUST_CLOSE_OPPOSING`); it runs against the wrong position. That is why this is one report. Stated plainly: symptoms 2-4 are on the same-side path, which ran the same live-position preview before `88ef3954` as well. They are the part of MED-14's remediation that was not done. Symptom 1 is the new code at line 437.

No value is lost. Each admitted invalid open reserves an execution bounty from the committer's own balance, and the keeper that clears the order is paid that bounty. A refused trader can queue the re-open after the close executes. The limits are stated in the Impact section.

## Relation to prior findings

**Cecuro MED-14** (Acknowledged, "Addressed"), "Trader can be blocked from queueing an open behind their closing order". Cecuro's remediation: "Base commit-time predictable-open rejection on the router's queued-position projection instead of the live engine position, **or** skip the MUST_CLOSE_OPPOSING pre-reject when earlier queued orders can remove the opposite-side exposure before this open reaches the head." The maintainers implemented the second option: "The live-position preview's MUST_CLOSE_OPPOSING result is ignored only after the queued-position side check passes." The rest of the screen still reads the live position. The four symptoms are the consequences of that choice:
- Skipping the `MUST_CLOSE_OPPOSING` result also skips everything the planner would have checked after it (symptom 1). The planner's early return hides those checks.
- The same-side case never reaches the skipped branch, so the live-position screen still decides it, in both directions: MED-14's impact remains (symptom 2), the live margin admits opens the projection shows invalid (symptom 3), and an earlier queued same-side open is ignored, so a valid increase is refused (symptom 4). MED-14's title and impact are not limited to one side, Cecuro's first remediation option ("Base commit-time predictable-open rejection on the router's queued-position projection instead of the live engine position") is exactly what all three of these need, and `ACCOUNTING_SPEC.md:1096` states the projection rule with no restriction to sides or to closes. The same-side code is unchanged by `88ef3954`, so symptoms 2-4 are filed as "the fix is incomplete", not as a regression.

**Cecuro MED-16 / LOW-57** (open items on the Scope tab: "opens committed against a **stale** cached price, and the keeper bounty sized from it"). MED-16's root is the freshness gate: with a stale mark, no planner screening runs, and the maintainers accept that ("Stale nonzero-mark admission is deliberately retained, with its bounty limitation accepted under LOW-57"). Symptoms 1 and 3 need a **fresh** mark (the PoC asserts the mark is under 60 s old, and the identical orders from a flat account are refused at that mark). Symptom 1 is caused by the `revertCode != opposingCode` clause added in `88ef3954`: before that commit, the same order with a fresh mark was screened and refused. Symptom 3 is caused by the screen planning on the live position, which Cecuro's MED-14 remediation asked to change. The MED-16 comment itself says "Price-dependent checks remain freshness-gated". This report is about a path where they are not.

Other nearby text, and why it is different:
- The MED-14 comment says "actual position, margin and risk checks run again at execution, so a failed earlier close can still make the later open fail". That accepts an open failing because the **earlier close** failed. In symptom 1 the close executes and the open is invalid regardless of it.
- `SECURITY.md:1353-1355` ("ordinary failed closes require unpaid cleanup … bounded deadlines and queues do not guarantee a positive keeper incentive or prevent repeated fresh invalid submissions") is about failed **closes**. It does not describe the open-admission screen skipping its checks with a fresh mark.
- Pre-audit SC-M2 ("Valid Same-side Increases are Rejected as Dust Orders") has a similar user-facing symptom to symptom 2 but a different root (incremental instead of resulting notional in the `POSITION_TOO_SMALL` check, since fixed). Symptom 2 is `SKEW_TOO_HIGH` on a position that a queued close removes; symptom 4 is `INSUFFICIENT_INITIAL_MARGIN` because an earlier queued open's size and margin are ignored.
- Pre-audit SC-M3 ("Fresh negative vpiAccrued is omitted from post-open IMR projection") is also an initial-margin admission gap, with a different root (the VPI liability is not projected; since fixed). Symptom 3 is the margin of a position that a queued close removes being counted.
- The Cantina and collaborative audit reports and the AI scan contain nothing on this path. The AI scan's stale-mark item (the MED-16 root) quotes the pre-fix line `if (commitMarkTime != 0)`.

## Vulnerability details

`_validatedOpenBounty` (`OrderLiquidationBatchLogic.sol:423-446`), reached from every open commit (`OrderLiquidationBatchLogic.sol:239`):

```solidity
        // Project the live position through earlier account orders before checking the requested side.
        QueuedPositionView memory queuedPosition = _queuedPosition(host, engine, account);
        uint8 opposingCode = uint8(CfdEnginePlanTypes.OpenRevertCode.MUST_CLOSE_OPPOSING);
        if (queuedPosition.exists && queuedPosition.side != side) {
            revert OrderRouter__PredictableOpenInvalid(opposingCode);
        }

        uint64 commitMarkTime = engine.lastMarkTime();
        uint8 revertCode =
            host.engineLensForCommit().previewOpenRevertCode(account, side, sizeDelta, marginDelta, commitPrice);
        if (revertCode == uint8(CfdEnginePlanTypes.OpenRevertCode.NUMERIC_DOMAIN_EXCEEDED)) {
            revert OrderRouter__PredictableOpenInvalid(revertCode);
        }
        // A queued close can remove an opposing live position before this open reaches the FIFO head.
        if (commitMarkTime != 0 && revertCode != opposingCode) {
            IPletherOracle.PolicySnapshot memory policy = host.pletherOracle().getOrderExecutionPolicy(false);
            if (!OracleFreshnessPolicyLib.isStale(commitMarkTime, policy.maxStaleness, block.timestamp)) {
                CfdEnginePlanTypes.OpenFailurePolicyCategory failureCategory =
                    CfdEnginePlanLib.getOpenFailurePolicyCategory(CfdEnginePlanTypes.OpenRevertCode(revertCode));
                if (OrderFailurePolicyLib.isPredictablyInvalidOpen(failureCategory)) {
                    revert OrderRouter__PredictableOpenInvalid(revertCode);
                }
            }
        }
```

`queuedPosition` is used only in the side check. `previewOpenRevertCode(account, …)` (`CfdEngineLens.sol:145-153`) goes to `_previewOpen`, which builds the snapshot from the account's **live** position (`CfdEngineLens.sol:262`, `_buildRawSnapshot(account, …)`) and plans the open on it. It knows nothing about the account's queued orders (closes or opens).

**Symptom 1, opposing open.** In `CfdEnginePlanLib.planOpen`, the opposite-side test (`CfdEnginePlanLib.sol:313-316`) comes before the degraded-mode test, before `_planOpenState` (which sets `SKEW_TOO_HIGH`, `CfdEnginePlanLib.sol:478`), and before the solvency and initial-margin checks that follow it (`CfdEnginePlanLib.sol:422`, `:435`):

```solidity
        if (effectiveSnap.position.size > 0 && effectiveSnap.position.side != order.side) {
            delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.MUST_CLOSE_OPPOSING;
            return delta;
        }
```

So whenever the live position is opposite, `revertCode` is `MUST_CLOSE_OPPOSING`, whatever the order's size or margin. Line 437 then skips the whole fresh-mark screen for that code. The projection has already passed the side check, so nothing else checks the order. The commit reserves the bounty and the order enters the global FIFO. At execution, after the close has run, the planner evaluates the open properly and it fails.

**Symptom 2, same-side open.** The live position is on the same side, so the planner runs in full on live + new. With LONG 300k live, a full close queued and LONG 200k requested, the planner checks the side skew for 500k against a cap of about 400.8k (the fixture's `maxSkewRatio` 0.4 × pool assets 1,002,000 in the PoC) and returns `SKEW_TOO_HIGH`. The mark is fresh, so line 443 refuses the commit. The position the planner sized never exists: the queued close executes first. The same shape gives `INSUFFICIENT_INITIAL_MARGIN` or `SOLVENCY_EXCEEDED` when live + new breaches those limits instead.

**Symptom 3, same-side open, admitting direction.** The same live + new plan also counts the live position's margin (`ACCOUNTING_SPEC.md:1092`: "Admission then checks the resulting pledge plus same-account claim …"). With LONG $10k live (400 USDC margin, ~386 after the open, mark 0.972), a full close queued and LONG $10k requested with 14 USDC margin, the planner checks a $20k position backed by ~400 USDC and returns `OK`. Line 443 has nothing to refuse. The position that will actually exist when the open reaches the head is a flat account opening $10k on 14 USDC, which the planner refuses with `INSUFFICIENT_INITIAL_MARGIN`: the PoC shows that code for a flat account at commit, and for the same trader once the close has executed. The open is admitted with a fresh mark and fails at execution. Line 437 plays no part here: the code is `OK`, so restoring the line to `if (commitMarkTime != 0)` would leave symptom 3 in place.

**Symptom 4, same-side open behind a queued open.** `_queuedPosition` does add earlier queued opens to the projection, but only the side check reads it. A flat trader with a queued SHORT $7.1k (1,491 USDC margin) who commits SHORT $23.8k with 238 USDC margin is planned as a flat account opening $23.8k on 238 USDC: `INSUFFICIENT_INITIAL_MARGIN` (1.5% of $23.8k is 357 USDC). The position that will exist when the second order reaches the head is SHORT $30.9k with ~1,729 USDC, against an initial margin of ~464 USDC. Once the first open has executed, the lens reads `OK` for the identical order and it fills.

The PoC's root-cause tests read the router's own commit lens (`router.engineLensForCommit()`) behind the queued full close: the $1M SHORT reads `MUST_CLOSE_OPPOSING` for the trader with the close and `SKEW_TOO_HIGH` for a flat account, the 200k LONG re-open reads `SKEW_TOO_HIGH`, and in the small-position setup the 14-USDC LONG reads `OK` for the trader with the close and `INSUFFICIENT_INITIAL_MARGIN` for a flat account and for the same trader after the close. Behind the queued open, the $23.8k SHORT reads `INSUFFICIENT_INITIAL_MARGIN`, and `OK` once that open has filled.

## Impact

**Who:** any trader, through ordinary actions (commit a close or an open, then commit another open). No role is needed.

**What the contract fails to deliver:**
- Symptom 1: the fresh-mark admission screen that the MED-16 remediation keeps ("Price-dependent checks remain freshness-gated") and that `ACCOUNTING_SPEC.md:1092-1094` describes ("Admission then checks the resulting pledge … the pool's post-trade settlement buffer. An order above the configured skew cap is allowed only when …"). Behind its own queued close, an account can place orders the router can already prove invalid into the global FIFO while the mark is fresh. In the PoC, three zero-margin $1M SHORTs are admitted (four orders pending with the close), and each one fails at execution after the close fills.
- Symptom 2: the MED-14 promise and `ACCOUNTING_SPEC.md:1096`. A trader cannot pre-queue a same-side re-open behind their own full close when live + new would breach a commit-time limit. They have to wait for the close to execute.
- Symptom 3: the same fresh-mark screen and Cecuro's MED-14 remediation ("Base commit-time predictable-open rejection on the router's queued-position projection instead of the live engine position"). Behind its own queued close, an account can place a same-side open whose margin cannot support it into the global FIFO while the mark is fresh. In the PoC it is admitted at 14 USDC margin on ~$9.7k and fails at execution after the close fills.
- Symptom 4: `ACCOUNTING_SPEC.md:1096` again. A trader cannot build a position over several queued same-side opens when a later order is only valid together with the earlier ones. They have to wait for each earlier open to execute.

This matches the program's Low impact "Contract fails to deliver promised returns, but doesn't lose value".

**Limits, stated plainly:**
- No value moves. Each admitted invalid open reserves an execution bounty from the committer's own balance (0.2 USDC each in the PoC), and the keeper that executes the failed order is paid it (the PoC logs the keeper receiving 0.6 USDC for three symptom-1 opens and 0.2 USDC for the symptom-3 open). Nothing is taken from other traders or from the pool.
- The per-account pending-order cap bounds symptoms 1 and 3: one slot holds the close, the rest can hold invalid opens. The failed orders are cleared by ordinary keeper execution.
- An account can already fill its pending slots with orders that will fail at execution (for example, opens whose slippage bound no price meets), and the stale-mark variant is an accepted limitation (LOW-57). Symptoms 1 and 3 are filed as the admission screen not doing what the fix and its comment say it does, not as a new way to load the queue.
- Symptoms 2 and 4 last until the trader's earlier queued order executes. The refused trader can queue the order as soon as that happens (shown in the PoC). Only the trader's own orders are affected.


## Recommendation

Feed the queued-position projection into the price-dependent preview, as Cecuro's MED-14 remediation says ("Base commit-time predictable-open rejection on the router's queued-position projection instead of the live engine position"). Run the screen on the position that will exist when the new open reaches the head: the live position after **all** of the account's earlier queued orders, closes and opens, including their **margin**. `_queuedPosition` tracks only size and side, and symptoms 3 and 4 are margin results. One way, as a sketch:

```solidity
// CfdEngineLens: classify an open as if the account's earlier queued orders had already executed at `oraclePrice`.
// Build the raw snapshot, then apply each earlier order in FIFO order with the planner's own plans (a close: the
// close plan for its size, which reduces the position and releases its margin; an open: the open plan, which adds
// size and margin; an order whose own plan fails is skipped, as it would not change the position at execution),
// updating the position and the side totals. Finally plan the new open on the result.
struct QueuedOrderView {
    bool isClose;
    CfdTypes.Side side;
    uint256 sizeDelta;
    uint256 marginDelta;
}

function previewOpenRevertCodeOnProjection(
    address account,
    QueuedOrderView[] calldata earlierOrders,
    CfdTypes.Side side,
    uint256 sizeDelta,
    uint256 marginDelta,
    uint256 oraclePrice
) external view returns (uint8 code);

// OrderLiquidationBatchLogic._validatedOpenBounty
QueuedOrderView[] memory earlier = _queuedOrders(host, account); // same walk and risk-off filter as _queuedPosition
uint8 revertCode = earlier.length == 0
    ? lens.previewOpenRevertCode(account, side, sizeDelta, marginDelta, commitPrice)
    : lens.previewOpenRevertCodeOnProjection(account, earlier, side, sizeDelta, marginDelta, commitPrice);
...
if (commitMarkTime != 0) { // no `revertCode != opposingCode` exemption
```

Planned on the projection, the planner no longer stops at `MUST_CLOSE_OPPOSING` behind a full close: it evaluates the opposing open's skew, solvency and margin (symptom 1). It sizes a same-side open on the projected position and against the margin that position will actually hold (symptoms 2, 3 and 4). A partial close that leaves opposing exposure is still refused by the existing side check. The walk is bounded by the per-account pending-order cap. As with every commit-time screen, this is an estimate (each earlier order settles at its own execution price), and execution still re-checks everything.

**Two narrower fixes do not work:**
- **Restoring line 437 to `if (commitMarkTime != 0)`**, the evident fix for symptom 1, refuses the opposing open behind a full close again, which undoes the MED-14 fix. It also leaves symptom 3 in place: there the screen already runs and returns `OK`, because it runs against the wrong position.
- **Feeding only the projected size** (what `_queuedPosition` already computes) into the preview addresses the skew cases (symptoms 1 and 2) but not the margin ones (symptoms 3 and 4). The projection has to carry margin as well.

Regression tests like the four `test_Property_` tests above would cover both directions. The current suite checks only that the opposing open is admitted.


## Link to Proof of Concept

https://gist.github.com/Richter2552/7be80af8467ac467f0f5985d5a981081

## Proof of Concept

## Proof of Concept

The program requires PoCs to run in the repository's Foundry test suite at the commit in scope without modifying the contracts in scope. The test file below extends the maintainers' `PositionProtectionTest` fixture. After the fixture's `setUp` it uses only public entry points from unprivileged accounts: production `OrderRouter.commitOrder(OrderRequest)` from the trader's own account, `router.executeOrder` from an address with no role, the permissionless `router.updateMarkPrice`, `HousePool.settleLpEpoch`, and `TrancheVault` request/claim. It also reads `previewOpenRevertCode` on the router's commit lens. Pyth is the maintainers' `MockPyth` (Pyth is outside the scope). No protocol contract is pranked and no storage is written. The fixture's junior deposit helper is switched off (`_initialJuniorDeposit() = 0`) because it impersonates the router. The junior LP deposits through the public vault flow instead. Because of that override, the tests inherited from `PositionProtectionTest` are not meant to run in this contract, so run only the tests named below.

Each **property** test states the behaviour the code should have and **fails** on the pinned code. Each **characterize/control** test passes and pins the exact behaviour.

Run instructions (from a checkout of `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with submodules; solc 0.8.35 via the repository's `packages/perps/foundry.toml`):

```bash
cp Card8785QueuedClosePreview.t.sol packages/perps/test/perps/spec/trader/
cd packages/perps

forge test --match-contract '^Card8785QueuedClosePreviewTest$' --match-test '^test_Property_' -vv

forge test --match-contract '^Card8785QueuedClosePreviewTest$' --match-test '^test_(Control|Characterize)_' -vv
```

### PoC output (pinned code, unmodified contracts)

Run against the pinned commit with the file installed next to `PositionProtection.t.sol` (forge 1.8.1, solc 0.8.35, the repository's `packages/perps/foundry.toml`). Result: the 8 characterize/control tests pass, and all four property tests fail. Excerpt (forge `-vv` logs; codes: 0 = `OK`, 1 = `MUST_CLOSE_OPPOSING`, 4 = `SKEW_TOO_HIGH`, 6 = `INSUFFICIENT_INITIAL_MARGIN`; USDC amounts in 1e6 units):

```console
[FAIL: next call did not revert as expected] test_Property_FreshMarkRejectsProvablyInvalidOpenBehindClose() (gas: 8443303)
[FAIL: next call did not revert as expected] test_Property_FreshMarkRejectsThinMarginSameSideOpenBehindClose() (gas: 8643856)
[FAIL: OrderRouter__PredictableOpenInvalid(6)] test_Property_SameSideIncreaseBehindQueuedOpenAdmitted() (gas: 4180742)
[FAIL: OrderRouter__PredictableOpenInvalid(4)] test_Property_SameSideReopenBehindFullCloseAdmitted() (gas: 7373897)

[PASS] test_Characterize_InvalidOpensAdmittedThenFailAtExecution() (gas: 14110574)
Logs:
  pool totalAssets        1002000000000
  max skew ~ 0.4 x depth  400800000000
  DAVE pending orders               4
  DAVE reserved bounty (3 opens)    600000
  keeper bounty from 3 failed opens 600000

[PASS] test_Characterize_RootCause_PreviewReadsLivePositionBehindFullClose() (gas: 7326797)
Logs:
  pool totalAssets        1002000000000
  max skew ~ 0.4 x depth  400800000000
  preview DAVE SHORT 1M  (behind full close) 1
  preview EVE  SHORT 1M  (flat)              4
  preview DAVE LONG 200k (behind full close) 4

[PASS] test_Characterize_SameSideIncreaseBehindQueuedOpenRefusalThenFill() (gas: 9730243)
Logs:
  preview EVE SHORT 23.8k @ 238 USDC (behind queued open) 6
  preview EVE SHORT 23.8k @ 238 USDC (after it filled)   0

[PASS] test_Characterize_SameSideRefusalAndValidityAfterClose() (gas: 11487043)
[PASS] test_Characterize_ThinMarginSameSideOpenAdmittedThenFailsAtExecution() (gas: 11472206)
Logs:
  preview DAVE LONG 10k @ 14 USDC  (behind full close) 0
  preview EVE  LONG 10k @ 14 USDC  (flat)              6
  preview EVE  LONG 10k @ 160 USDC (flat)              0
  preview DAVE LONG 10k @ 14 USDC  (after the close)   6
  keeper bounty from the failed open   200000

[PASS] test_Control_FreshMarkRejectsOversizedOpenFromFlat() (gas: 7048728)
[PASS] test_Control_FreshMarkRejectsThinMarginOpenFromFlat() (gas: 7367118)
[PASS] test_Control_OpposingOpenBehindFullCloseAdmitted() (gas: 8116734)
Suite result: ok. 8 passed; 0 failed; 0 skipped
```

Expected output on a fixed contract: all four property tests pass. The four `test_Characterize_` tests other than the root-cause test then fail, because the admissions and refusals they pin no longer happen. The root-cause test also fails, because it pins the live-position preview codes.

### What the tests show

| step | test | result on the pinned code |
|---|---|---|
| DAVE is LONG 300k (opened through `commitOrder` + keeper `executeOrder`) and queues a full close. The router's commit lens still classifies his orders against LONG 300k: a $1M SHORT reads `MUST_CLOSE_OPPOSING` (a flat account reads `SKEW_TOO_HIGH`), and a 200k LONG re-open reads `SKEW_TOO_HIGH` | `test_Characterize_RootCause_PreviewReadsLivePositionBehindFullClose` | passes |
| control: with a fresh mark, a flat account's $1M SHORT is refused at commit, `PredictableOpenInvalid(SKEW_TOO_HIGH)` | `test_Control_FreshMarkRejectsOversizedOpenFromFlat` | passes |
| symptom 1: the same $1M SHORT from DAVE, behind his queued close, with the mark under 60 s old, should be refused the same way | `test_Property_FreshMarkRejectsProvablyInvalidOpenBehindClose` | **FAILS**: the order is admitted |
| symptom 1: three zero-margin $1M SHORTs are admitted (four orders pending), the close executes, each SHORT fails at execution, and the keeper receives the three reserved bounties | `test_Characterize_InvalidOpensAdmittedThenFailAtExecution` | passes |
| control: what the MED-14 fix covers, an opposing 200k open behind the full close, is admitted | `test_Control_OpposingOpenBehindFullCloseAdmitted` | passes |
| symptom 2: DAVE queues LONG 200k (20k margin) behind his full close | `test_Property_SameSideReopenBehindFullCloseAdmitted` | **FAILS**: `OrderRouter__PredictableOpenInvalid(4)` (`SKEW_TOO_HIGH`) |
| symptom 2: the exact refusal; after the close executes, the identical open is admitted and fills | `test_Characterize_SameSideRefusalAndValidityAfterClose` | passes |
| symptom 3 setup: DAVE is LONG $10k (400 USDC margin, opened through the router), anyone refreshes the mark to 0.972, and DAVE queues a full close. Control: with that fresh mark, flat EVE's LONG $10k with 14 USDC margin is refused at commit, `PredictableOpenInvalid(INSUFFICIENT_INITIAL_MARGIN)` | `test_Control_FreshMarkRejectsThinMarginOpenFromFlat` | passes |
| symptom 3: the same 14-USDC LONG from DAVE, behind his queued close, should be refused the same way | `test_Property_FreshMarkRejectsThinMarginSameSideOpenBehindClose` | **FAILS**: the order is admitted |
| symptom 3: the commit lens reads `OK` for DAVE and `INSUFFICIENT_INITIAL_MARGIN` for EVE (160 USDC reads `OK`); DAVE's order is admitted, the close executes, the lens now reads `INSUFFICIENT_INITIAL_MARGIN` for DAVE, the admitted open ends `Failed`, and the keeper receives the 0.2 USDC bounty | `test_Characterize_ThinMarginSameSideOpenAdmittedThenFailsAtExecution` | passes |
| symptom 4: flat EVE queues SHORT $7.1k (1,491 USDC margin), which stays queued, then commits SHORT $23.8k (238 USDC), valid on the projected $30.9k | `test_Property_SameSideIncreaseBehindQueuedOpenAdmitted` | **FAILS**: `OrderRouter__PredictableOpenInvalid(6)` (`INSUFFICIENT_INITIAL_MARGIN`) |
| symptom 4: the exact refusal (lens code 6 behind the queued open); the first open executes, the lens reads `OK`, and the identical request is admitted and fills (SHORT $30.9k) | `test_Characterize_SameSideIncreaseBehindQueuedOpenRefusalThenFill` | passes |

### Test file: `packages/perps/test/perps/spec/trader/Card8785QueuedClosePreview.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineLens} from "@plether/perps/interfaces/ICfdEngineLens.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title Open admission behind a queued full close classifies the open against the LIVE position (MED-14 fix)
/// @notice Pinned commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. Install next to PositionProtection.t.sol in
///         packages/perps/test/perps/spec/trader/ and run with the repository's own foundry.toml.
/// @dev `_validatedOpenBounty` (router/OrderLiquidationBatchLogic.sol:404-455) projects the account's live position
///      through its earlier queued orders (`_queuedPosition`, :367-402), but uses the projection ONLY for the side
///      check (:424-428). The price-dependent screen then calls `previewOpenRevertCode(account, ...)` (:431-432), which
///      plans the open on the LIVE engine position, ignoring the account's earlier queued orders. Four symptoms follow:
///      - opposing open (live position on the other side): the planner returns MUST_CLOSE_OPPOSING
///        (CfdEnginePlanLib.sol:313-316) before skew, solvency or margin, and :437 then skips the whole fresh-mark
///        screen, so an open the router can prove invalid is admitted with a FRESH mark;
///      - same-side open: the planner sizes live + new, a position that never exists, so an open that is valid on its
///        own is refused (SKEW_TOO_HIGH here) until the close executes;
///      - same-side open, other direction: the live position's margin carries an open whose own margin is far too
///        small, so an open the router refuses from a flat account (INSUFFICIENT_INITIAL_MARGIN) is admitted with a
///        FRESH mark and fails at execution;
///      - same-side open behind a queued same-side OPEN: the queued open's size and margin are ignored, so an
///        increase that is valid on the projected position is refused (INSUFFICIENT_INITIAL_MARGIN here).
///      Public entry points only after the fixture's setUp: V3 `OrderRouter.commitOrder(OrderRequest)` from the
///      trader's own account, `router.executeOrder` from KEEPER (no role), the permissionless `router.updateMarkPrice`,
///      `HousePool.settleLpEpoch` and `TrancheVault` request/claim. Pyth is the maintainers' MockPyth. No protocol
///      contract is pranked and no storage is written. `_initialJuniorDeposit()` is 0 because the fixture's helper
///      impersonates the router, so the tests inherited from PositionProtectionTest are not meant to run here.
contract Card8785QueuedClosePreviewTest is PositionProtectionTest {

    address internal constant LP = address(0x1B1B);
    address internal constant KEEPER = address(0x5EE5); // no role anywhere; any EOA
    address internal constant REFRESHER = address(0x4EF4); // any EOA
    address internal constant DAVE = address(0xDA7E); // ordinary trader
    address internal constant EVE = address(0xE7E); // ordinary trader, flat
    uint256 internal constant JUNIOR_LP_ASSETS = 1_000_000e6;
    uint256 internal constant TRADER_FUNDS = 200_000e6;

    uint256 internal constant FIRST_SIZE = 300_000e18; // $300k notional at MARK_PRICE = 1e8 ($1)
    uint256 internal constant FIRST_MARGIN = 30_000e6;
    uint256 internal constant REOPEN_SIZE = 200_000e18; // $200k: valid alone; 300k + 200k > max skew (~$400.8k)
    uint256 internal constant REOPEN_MARGIN = 20_000e6;
    uint256 internal constant HUGE_SHORT = 1_000_000e18; // $1M: more than twice max skew, invalid at any fill

    uint8 internal constant OK = uint8(CfdEnginePlanTypes.OpenRevertCode.OK);
    uint8 internal constant SKEW_TOO_HIGH = uint8(CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH);
    uint8 internal constant MUST_CLOSE_OPPOSING = uint8(CfdEnginePlanTypes.OpenRevertCode.MUST_CLOSE_OPPOSING);

    uint256 private _clientNonce;

    /// No junior deposit in the fixture setUp: its helper pranks the router into `engine.updateMarkPrice`.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Public-entry-point helpers
    // ---------------------------------------------------------------------------------------------------------------

    /// A live Pyth tick at the current block time, chained to the last engine mark time.
    function _tickNow(
        uint256 price
    ) private view returns (bytes[] memory updateData) {
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

    /// Anyone installs a fresh accepted basket via the permissionless `router.updateMarkPrice`.
    function _publicRefresh(
        address caller,
        uint256 price
    ) private {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _tickNow(price);
        vm.prank(caller);
        router.updateMarkPrice(data);
    }

    /// An LP funds junior through the vault's public request/claim flow. The epoch is settled permissionlessly.
    function _publicJuniorDeposit(
        address lp,
        uint256 amount
    ) private {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();

        uint256 activationTime = juniorVault.depositEpochStart(requestId);
        if (vm.getBlockTimestamp() < activationTime) {
            vm.warp(activationTime);
        }
        _publicRefresh(lp, MARK_PRICE);
        vm.prank(lp);
        pool.settleLpEpoch(0, 0);

        uint256 claimable = juniorVault.claimableDepositRequest(requestId, lp);
        assertEq(claimable, amount, "ghost: LP deposit finalized");
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, claimable, lp, lp);
    }

    /// A production V3 order request (slippage bound never binding).
    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) private returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("card-8785", ++_clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
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
            })
        });
    }

    function _commitV3(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) private returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _request(side, size, margin, isClose);
        vm.prank(trader);
        orderId = OrderRouter(address(router)).commitOrder(request);
    }

    /// KEEPER (no role) executes with the first Pyth tick after commitTime.
    function _keeperExecuteAt(
        uint64 orderId,
        uint256 price
    ) private returns (OrderV3Types.ExecutionResult memory res) {
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        uint64 committedAt = pending.commitTime;
        if (vm.getBlockTimestamp() < uint256(committedAt) + 2) {
            vm.warp(uint256(committedAt) + 2);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _mockUniquePythUpdateDataAt(price, committedAt + 1, committedAt);
        vm.prank(KEEPER);
        res = router.executeOrder(orderId, data);
    }

    function _keeperExecute(
        uint64 orderId
    ) private {
        OrderV3Types.ExecutionResult memory res = _keeperExecuteAt(orderId, MARK_PRICE);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: order executed by keeper");
    }

    /// `trader` opens through commit -> keeper execute, after a public refresh.
    function _openThroughRouter(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) private {
        vm.warp(vm.getBlockTimestamp() + 10);
        _publicRefresh(REFRESHER, MARK_PRICE);
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        uint64 orderId = _commitV3(trader, side, size, margin, false);
        _keeperExecute(orderId);
    }

    function _bootstrapPool() private {
        assertTrue(pool.isTradingActive(), "ghost: trading activated");
        _publicJuniorDeposit(LP, JUNIOR_LP_ASSETS);
        _fundTrader(DAVE, TRADER_FUNDS); // DAVE's own USDC through clearinghouse.deposit
        _fundTrader(EVE, TRADER_FUNDS);
    }

    /// DAVE holds LONG 300k opened through the router, then queues a FULL close of it. The mark is fresh.
    function _daveLongThenQueuedFullClose() private returns (uint64 closeId) {
        _bootstrapPool();
        _openThroughRouter(DAVE, CfdTypes.Side.LONG, FIRST_SIZE, FIRST_MARGIN);
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(DAVE);
        assertEq(size, FIRST_SIZE, "ghost: DAVE holds LONG 300k via the router");
        assertEq(uint8(side), uint8(CfdTypes.Side.LONG));

        closeId = _commitV3(DAVE, CfdTypes.Side.LONG, FIRST_SIZE, 0, true);
        assertEq(router.pendingCloseSize(DAVE), FIRST_SIZE, "ghost: full close queued");
        vm.warp(vm.getBlockTimestamp() + 1);
        assertLt(vm.getBlockTimestamp() - engine.lastMarkTime(), 60, "ghost: cached mark is fresh");
        console2.log("pool totalAssets       ", pool.totalAssets());
        console2.log("max skew ~ 0.4 x depth ", (pool.totalAssets() * 4) / 10);
    }

    function _preview(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) private view returns (uint8) {
        return ICfdEngineLens(OrderRouter(address(router)).engineLensForCommit())
            .previewOpenRevertCode(account, side, size, margin, engine.lastMarkPrice());
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Root cause: the commit-time preview classifies the open against the LIVE position, not the projected one
    // ---------------------------------------------------------------------------------------------------------------

    /// CHARACTERIZE: behind DAVE's queued full close the router's preview still sees LONG 300k. The SHORT that the
    /// same preview rejects for a flat account (SKEW_TOO_HIGH) reads MUST_CLOSE_OPPOSING for DAVE; the LONG re-open
    /// that becomes valid once the close runs reads SKEW_TOO_HIGH for DAVE.
    function test_Characterize_RootCause_PreviewReadsLivePositionBehindFullClose() public {
        _daveLongThenQueuedFullClose();
        uint8 daveShort = _preview(DAVE, CfdTypes.Side.SHORT, HUGE_SHORT, REOPEN_MARGIN);
        uint8 eveShort = _preview(EVE, CfdTypes.Side.SHORT, HUGE_SHORT, REOPEN_MARGIN);
        uint8 daveLong = _preview(DAVE, CfdTypes.Side.LONG, REOPEN_SIZE, REOPEN_MARGIN);
        console2.log("preview DAVE SHORT 1M  (behind full close)", daveShort);
        console2.log("preview EVE  SHORT 1M  (flat)             ", eveShort);
        console2.log("preview DAVE LONG 200k (behind full close)", daveLong);
        assertEq(daveShort, MUST_CLOSE_OPPOSING, "DAVE's opposing open is classified against the live LONG");
        assertEq(eveShort, SKEW_TOO_HIGH, "the same SHORT from a flat account is SKEW_TOO_HIGH");
        assertEq(daveLong, SKEW_TOO_HIGH, "DAVE's same-side re-open is sized on top of the live LONG 300k");
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Symptom 1 (new bug, :437): an opposing open the router can prove invalid is admitted with a FRESH mark
    // ---------------------------------------------------------------------------------------------------------------

    /// CONTROL: with a fresh mark and no queued close, the router refuses this exact open at commit (SKEW_TOO_HIGH).
    function test_Control_FreshMarkRejectsOversizedOpenFromFlat() public {
        _daveLongThenQueuedFullClose();
        OrderV3Types.OrderRequest memory req = _request(CfdTypes.Side.SHORT, HUGE_SHORT, REOPEN_MARGIN, false);
        vm.prank(EVE);
        vm.expectRevert(abi.encodeWithSelector(IOrderRouterErrors.OrderRouter__PredictableOpenInvalid.selector, SKEW_TOO_HIGH));
        OrderRouter(address(router)).commitOrder(req);
    }

    /// PROPERTY (MED-16 remediation: "Price-dependent checks remain freshness-gated"): with a fresh mark, the same
    /// provably invalid open is refused at commit when it sits behind the account's own queued full close.
    /// FAILS on the pinned code: the order is admitted.
    function test_Property_FreshMarkRejectsProvablyInvalidOpenBehindClose() public {
        _daveLongThenQueuedFullClose();
        OrderV3Types.OrderRequest memory req = _request(CfdTypes.Side.SHORT, HUGE_SHORT, REOPEN_MARGIN, false);
        vm.prank(DAVE);
        vm.expectRevert(abi.encodeWithSelector(IOrderRouterErrors.OrderRouter__PredictableOpenInvalid.selector, SKEW_TOO_HIGH));
        OrderRouter(address(router)).commitOrder(req);
    }

    /// CHARACTERIZE: with the close queued and a fresh mark, three zero-margin $1M SHORTs are admitted into the FIFO,
    /// the close executes, and every one of them then fails at execution. Logs the bounties.
    function test_Characterize_InvalidOpensAdmittedThenFailAtExecution() public {
        uint64 closeId = _daveLongThenQueuedFullClose();
        uint64[3] memory ids;
        for (uint256 i; i < 3; ++i) {
            ids[i] = _commitV3(DAVE, CfdTypes.Side.SHORT, HUGE_SHORT, 0, false); // zero margin as well
            assertGt(ids[i], 0, "admitted");
        }
        assertEq(router.pendingOrderCounts(DAVE), 4, "close + 3 admitted invalid opens pending");
        console2.log("DAVE pending orders              ", router.pendingOrderCounts(DAVE));
        console2.log("DAVE reserved bounty (3 opens)   ", router.getAccountReservations(DAVE).executionBountyAssets);

        _keeperExecute(closeId);
        (uint256 sizeAfterClose,,,,,,) = engine.positions(DAVE);
        assertEq(sizeAfterClose, 0, "ghost: the queued close executed; DAVE is flat");

        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        for (uint256 i; i < 3; ++i) {
            OrderV3Types.ExecutionResult memory res = _keeperExecuteAt(ids[i], MARK_PRICE);
            assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Failed), "admitted open fails at execution");
        }
        (uint256 sizeAfter,,,,,,) = engine.positions(DAVE);
        assertEq(sizeAfter, 0, "no admitted open ever filled");
        console2.log("keeper bounty from 3 failed opens", clearinghouse.balanceAssets(KEEPER) - keeperBefore);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Symptom 2 (fix incomplete): a same-side open that is valid on its own is refused behind the full close
    // ---------------------------------------------------------------------------------------------------------------

    /// CONTROL (what the MED-14 fix covers): the OPPOSING open behind the same full close is admitted.
    function test_Control_OpposingOpenBehindFullCloseAdmitted() public {
        _daveLongThenQueuedFullClose();
        uint64 id = _commitV3(DAVE, CfdTypes.Side.SHORT, REOPEN_SIZE, REOPEN_MARGIN, false);
        assertGt(id, 0, "control: opposing open admitted (MED-14 fix)");
    }

    /// PROPERTY (MED-14: "Users cannot pre-queue the next open behind a closing order"; ACCOUNTING_SPEC.md:1096 "An
    /// open must match the position projected through earlier queued orders"): behind a queued full close, an open
    /// that is valid on its own can be queued. FAILS on the pinned code with PredictableOpenInvalid(SKEW_TOO_HIGH).
    function test_Property_SameSideReopenBehindFullCloseAdmitted() public {
        _daveLongThenQueuedFullClose();
        uint64 reopenId = _commitV3(DAVE, CfdTypes.Side.LONG, REOPEN_SIZE, REOPEN_MARGIN, false);
        assertGt(reopenId, 0, "re-open queued behind the full close");
    }

    /// CHARACTERIZE: the exact refusal, and that the identical open is admitted and fills once the close executes.
    function test_Characterize_SameSideRefusalAndValidityAfterClose() public {
        uint64 closeId = _daveLongThenQueuedFullClose();
        OrderV3Types.OrderRequest memory reopen = _request(CfdTypes.Side.LONG, REOPEN_SIZE, REOPEN_MARGIN, false);
        vm.prank(DAVE);
        vm.expectRevert(abi.encodeWithSelector(IOrderRouterErrors.OrderRouter__PredictableOpenInvalid.selector, SKEW_TOO_HIGH));
        OrderRouter(address(router)).commitOrder(reopen);

        _keeperExecute(closeId);
        (uint256 sizeAfterClose,,,,,,) = engine.positions(DAVE);
        assertEq(sizeAfterClose, 0, "ghost: DAVE flat after the queued close");

        _openThroughRouter(DAVE, CfdTypes.Side.LONG, REOPEN_SIZE, REOPEN_MARGIN);
        (uint256 sizeAfterReopen,,,,,,) = engine.positions(DAVE);
        assertEq(sizeAfterReopen, REOPEN_SIZE, "the refused open is valid and fills once the close has run");
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Symptom 3 (fix incomplete): a same-side open with too little margin is ADMITTED behind the full close, because
    // the live position's margin carries it in the preview
    // ---------------------------------------------------------------------------------------------------------------

    uint256 internal constant SMALL_SIZE = 10_000e18; // $10k, 1% of pool depth: skew is irrelevant
    uint256 internal constant SMALL_MARGIN = 400e6;
    uint256 internal constant SMALL_FUNDS = 2_000e6;
    uint256 internal constant DROP_PRICE = 0.972e8; // DAVE's LONG is down ~$280 but not liquidatable
    uint256 internal constant THIN_MARGIN = 14e6; // 14 USDC on ~$9.7k: far below the initial margin
    uint256 internal constant STANDALONE_MARGIN = 160e6; // valid alone at 0.972 (fixture IM 150 bps of ~$9.7k = ~$146)
    uint8 internal constant INSUFFICIENT_INITIAL_MARGIN =
        uint8(CfdEnginePlanTypes.OpenRevertCode.INSUFFICIENT_INITIAL_MARGIN);

    /// DAVE holds a small LONG $10k (400 USDC margin) opened through the router, the price drops to 0.972 (a fresh
    /// public mark), and DAVE queues a FULL close. EVE is flat and funded. The mark is fresh.
    function _daveSmallLongThenQueuedFullClose() private returns (uint64 closeId) {
        assertTrue(pool.isTradingActive(), "ghost: trading activated");
        _publicJuniorDeposit(LP, JUNIOR_LP_ASSETS);
        _fundTrader(DAVE, SMALL_FUNDS);
        _fundTrader(EVE, SMALL_FUNDS);
        _openThroughRouter(DAVE, CfdTypes.Side.LONG, SMALL_SIZE, SMALL_MARGIN);
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(DAVE);
        assertEq(size, SMALL_SIZE, "ghost: DAVE holds LONG $10k via the router");
        assertEq(uint8(side), uint8(CfdTypes.Side.LONG));

        vm.warp(vm.getBlockTimestamp() + 1);
        _publicRefresh(REFRESHER, DROP_PRICE);
        vm.warp(vm.getBlockTimestamp() + 1);
        closeId = _commitV3(DAVE, CfdTypes.Side.LONG, SMALL_SIZE, 0, true);
        assertEq(router.pendingCloseSize(DAVE), SMALL_SIZE, "ghost: full close queued");
        vm.warp(vm.getBlockTimestamp() + 1);
        assertEq(engine.lastMarkPrice(), DROP_PRICE, "ghost: mark is the dropped price");
        assertLt(vm.getBlockTimestamp() - engine.lastMarkTime(), 60, "ghost: cached mark is fresh");
    }

    /// CONTROL: with a fresh mark, a flat funded account's LONG $10k with 14 USDC margin is refused at commit,
    /// `PredictableOpenInvalid(INSUFFICIENT_INITIAL_MARGIN)`.
    function test_Control_FreshMarkRejectsThinMarginOpenFromFlat() public {
        _daveSmallLongThenQueuedFullClose();
        OrderV3Types.OrderRequest memory req = _request(CfdTypes.Side.LONG, SMALL_SIZE, THIN_MARGIN, false);
        vm.prank(EVE);
        vm.expectRevert(
            abi.encodeWithSelector(
                IOrderRouterErrors.OrderRouter__PredictableOpenInvalid.selector, INSUFFICIENT_INITIAL_MARGIN
            )
        );
        OrderRouter(address(router)).commitOrder(req);
    }

    /// PROPERTY (Cecuro MED-14 remediation "Base commit-time predictable-open rejection on the router's queued-position
    /// projection instead of the live engine position"; MED-16 "Price-dependent checks remain freshness-gated"): with
    /// a fresh mark, a same-side open whose margin cannot support it on the projected (flat) position is refused at
    /// commit, as it is for a flat account. FAILS on the pinned code: the order is admitted.
    function test_Property_FreshMarkRejectsThinMarginSameSideOpenBehindClose() public {
        _daveSmallLongThenQueuedFullClose();
        OrderV3Types.OrderRequest memory req = _request(CfdTypes.Side.LONG, SMALL_SIZE, THIN_MARGIN, false);
        vm.prank(DAVE);
        vm.expectRevert(
            abi.encodeWithSelector(
                IOrderRouterErrors.OrderRouter__PredictableOpenInvalid.selector, INSUFFICIENT_INITIAL_MARGIN
            )
        );
        OrderRouter(address(router)).commitOrder(req);
    }

    /// CHARACTERIZE: the router's commit lens classifies DAVE's thin-margin LONG as OK behind his queued close (the
    /// live LONG's margin carries it), the same open as INSUFFICIENT_INITIAL_MARGIN for flat EVE and for DAVE once the
    /// close has executed. The router admits DAVE's order, the close executes, and the admitted open fails at execution;
    /// the keeper is paid the reserved bounty. A standalone margin (160 USDC) is OK for everyone.
    function test_Characterize_ThinMarginSameSideOpenAdmittedThenFailsAtExecution() public {
        uint64 closeId = _daveSmallLongThenQueuedFullClose();
        uint8 daveThin = _preview(DAVE, CfdTypes.Side.LONG, SMALL_SIZE, THIN_MARGIN);
        uint8 eveThin = _preview(EVE, CfdTypes.Side.LONG, SMALL_SIZE, THIN_MARGIN);
        uint8 eveStandalone = _preview(EVE, CfdTypes.Side.LONG, SMALL_SIZE, STANDALONE_MARGIN);
        console2.log("preview DAVE LONG 10k @ 14 USDC  (behind full close)", daveThin);
        console2.log("preview EVE  LONG 10k @ 14 USDC  (flat)             ", eveThin);
        console2.log("preview EVE  LONG 10k @ 160 USDC (flat)             ", eveStandalone);
        assertEq(daveThin, OK, "DAVE's thin-margin open is carried by the live LONG's margin");
        assertEq(eveThin, INSUFFICIENT_INITIAL_MARGIN, "the same open from a flat account is refused");
        assertEq(eveStandalone, OK, "control: a standalone margin is admissible");

        uint64 openId = _commitV3(DAVE, CfdTypes.Side.LONG, SMALL_SIZE, THIN_MARGIN, false);
        assertGt(openId, 0, "thin-margin same-side open admitted behind the full close");

        OrderV3Types.ExecutionResult memory res = _keeperExecuteAt(closeId, DROP_PRICE);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: the queued close executed");
        (uint256 flat,,,,,,) = engine.positions(DAVE);
        assertEq(flat, 0, "ghost: DAVE is flat");
        uint8 daveAfter = _preview(DAVE, CfdTypes.Side.LONG, SMALL_SIZE, THIN_MARGIN);
        console2.log("preview DAVE LONG 10k @ 14 USDC  (after the close)  ", daveAfter);
        assertEq(daveAfter, INSUFFICIENT_INITIAL_MARGIN, "once the close has run, DAVE's open reads as for EVE");

        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        res = _keeperExecuteAt(openId, DROP_PRICE);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Failed), "admitted open fails at execution");
        (uint256 sizeAfter,,,,,,) = engine.positions(DAVE);
        assertEq(sizeAfter, 0, "the admitted open never filled");
        console2.log("keeper bounty from the failed open  ", clearinghouse.balanceAssets(KEEPER) - keeperBefore);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Symptom 4 (fix incomplete): behind the account's own queued same-side OPEN, a valid increase is refused, because
    // the preview ignores the queued open's size and margin
    // ---------------------------------------------------------------------------------------------------------------

    uint256 internal constant QUEUED_OPEN_SIZE = 7_100e18; // $7.1k
    uint256 internal constant QUEUED_OPEN_MARGIN = 1_491e6; // 21 %
    uint256 internal constant INCREASE_SIZE = 23_800e18; // $23.8k
    uint256 internal constant INCREASE_MARGIN = 238e6; // 1 %: too little alone (IM 1.5 %), ample on the combined $30.9k

    /// EVE (flat, funded) queues SHORT $7.1k with 21 % margin; it stays queued. The mark is fresh.
    function _eveQueuedSameSideOpen() private returns (uint64 firstId) {
        _bootstrapPool();
        vm.warp(vm.getBlockTimestamp() + 10);
        _publicRefresh(REFRESHER, MARK_PRICE);
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        firstId = _commitV3(EVE, CfdTypes.Side.SHORT, QUEUED_OPEN_SIZE, QUEUED_OPEN_MARGIN, false);
        assertEq(router.pendingOrderCounts(EVE), 1, "ghost: first open queued");
        (uint256 live,,,,,,) = engine.positions(EVE);
        assertEq(live, 0, "ghost: EVE's live position is still flat");
        vm.warp(vm.getBlockTimestamp() + 1);
        assertLt(vm.getBlockTimestamp() - engine.lastMarkTime(), 60, "ghost: cached mark is fresh");
    }

    /// PROPERTY (ACCOUNTING_SPEC.md:1096 "An open must match the position projected through earlier queued orders";
    /// Cecuro MED-14 remediation): behind the account's own queued SHORT $7.1k (21 % margin), a SHORT $23.8k at 1 %
    /// margin, valid on the projected $30.9k position, can be queued. FAILS on the pinned code with
    /// PredictableOpenInvalid(INSUFFICIENT_INITIAL_MARGIN): the preview plans it on the live (flat) position.
    function test_Property_SameSideIncreaseBehindQueuedOpenAdmitted() public {
        _eveQueuedSameSideOpen();
        uint64 id = _commitV3(EVE, CfdTypes.Side.SHORT, INCREASE_SIZE, INCREASE_MARGIN, false);
        assertGt(id, 0, "increase queued behind the earlier same-side open");
    }

    /// CHARACTERIZE: the exact refusal behind the queued open, and that the identical request is admitted and fills
    /// once the first open has executed (combined SHORT $30.9k).
    function test_Characterize_SameSideIncreaseBehindQueuedOpenRefusalThenFill() public {
        uint64 firstId = _eveQueuedSameSideOpen();
        uint8 code = _preview(EVE, CfdTypes.Side.SHORT, INCREASE_SIZE, INCREASE_MARGIN);
        console2.log("preview EVE SHORT 23.8k @ 238 USDC (behind queued open)", code);
        OrderV3Types.OrderRequest memory req = _request(CfdTypes.Side.SHORT, INCREASE_SIZE, INCREASE_MARGIN, false);
        vm.prank(EVE);
        vm.expectRevert(
            abi.encodeWithSelector(
                IOrderRouterErrors.OrderRouter__PredictableOpenInvalid.selector, INSUFFICIENT_INITIAL_MARGIN
            )
        );
        OrderRouter(address(router)).commitOrder(req);

        _keeperExecute(firstId);
        (uint256 afterFirst,,,,,,) = engine.positions(EVE);
        assertEq(afterFirst, QUEUED_OPEN_SIZE, "ghost: the first open filled");
        console2.log(
            "preview EVE SHORT 23.8k @ 238 USDC (after it filled)  ",
            _preview(EVE, CfdTypes.Side.SHORT, INCREASE_SIZE, INCREASE_MARGIN)
        );

        _openThroughRouter(EVE, CfdTypes.Side.SHORT, INCREASE_SIZE, INCREASE_MARGIN);
        (uint256 combined,,,,,,) = engine.positions(EVE);
        assertEq(combined, QUEUED_OPEN_SIZE + INCREASE_SIZE, "the refused increase fills once the queued open has run");
    }

}
```

