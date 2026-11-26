# \#99915 [SC-Low] Queued partial closes are not binding: a trader can decide after commit whether its own close executes (an equity bound with addMargin, or a free-settlement withdrawal on an ordinary close)

**Submitted on Nov 15th 2026 at 18:03:45 UTC by @argo_the_hacker for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99915
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

**Severity: Insight.** Filed under the program's Insight Reporting rules. No impact from the program's list is claimed; the Impact section says why. The protocol states that "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options" (`packages/perps/SECURITY.md:658-659`). At the pinned commit a trader with no role can still decide, after commit, whether its own queued partial close executes or fails terminally, at a fill already fixed by the first historical tick after commit. We show two levers. One uses an equity bound and `addMargin`. The other is an ordinary close with no unusual bound and a withdrawal of free settlement. A repeated-trade test measures what the choice is worth: 0.31 to 0.47 bp of each committed close, paid by LPs, under an assumed volatility. The same test shows the trader losing money net of fees. So this report claims a broken design guarantee with a measured, small value transfer, not an exploit.

**Target:** `immunefi-team/audit-comp-plether` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`:

- the commit-time request validation `OrderLiquidationBatchLogic._validateFreshRequest` (`OrderLiquidationBatchLogic.sol:270-319`), which never reads `minPostPositionEquityAssets`;
- the execution-time bound check (`CfdOrderPolicyEvaluatorBase.sol:530-539`) and its terminal classification (`OrderRouterExecutionSidecar.sol:879-891`);
- `CfdEngine.addMargin` (`CfdEngine.sol:501-527`);
- the partial-close charge collection and its terminal rejection (`CfdEnginePlanLib.sol:1092-1133`, `:1137-1180`, `:882-885`), and the owner withdrawal path (`MarginClearinghouse._withdraw`, `MarginClearinghouse.sol:441-477`; `CfdEngine.checkWithdraw`, `CfdEngine.sol:730-748`).

**Relation to prior findings:** this is the class of the pre-audit's SC-M4 "Risk Free Option in 60Sec Oracle Window" (fixed, Plether-Fi/plether-core#24). Both levers were added to the code after that fix. Under the program's rule we report it as **the SC-M4 fix being incomplete at `a82d4fef`**. It is a different root cause from Cecuro HIGH-1 (Arbitrum/Pyth clock alignment). HIGH-1 is about what a trader can *know* at commit. This report is about a *choice* the trader makes after commit, whatever the clock. See "Relation to prior findings" below.

## Summary

A trader with no role can decide, after commit, whether its own queued partial close executes or fails. This was executed on the pinned code with two levers:

- **Lever 1, equity bound and `addMargin`.** The trader commits a partial close with a `minPostPositionEquityAssets` floor that the residual cannot reach on its own. The same order at the same fill tick ends `Failed` (`ConstraintViolation`) if the trader does nothing, and `Executed` if the trader first calls `addMargin`. The same lever works on a position increase that the trader executes itself.
- **Lever 2, an ordinary close and a withdrawal.** The partial close carries no unusual bound. In a narrow band of losing fills, the same order at the same fill ends `Executed` if the trader leaves its free settlement in place. It ends `Failed` (planner rejection, `PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE`) if the trader first withdraws that settlement. At the release risk parameters the band is +0.315% to +0.330% of adverse move for the tested position.

In both levers the failed branch settles no close and leaves the position size unchanged. The failed attempt takes no USDG from the trader and pays no executor. It is not free in every sense: committing costs gas and collects carry, a self-executing trader pays gas and any oracle fee, and the open position keeps its market risk and carry.

What the tests prove, and what they do not:

- **They prove the switch.** At a fixed historical fill, the outcome depends on an owner action taken after commit. In the single-trade tests the mocked historical tick is constructed at execution, after the trader's action. So these tests do not stage the sequence "see the published fill, see a later market price, then choose". The strict status equality in the property tests is the requirement we propose. The code does not document it in that exact form.
- **The value of the choice is measured separately.** A repeated-trade test (lever 1, release parameters, the shipped 60 s window) models the trader deciding with the fill price F and a later price D. It closes every round flat, so each result is fully realised wealth. The choice is worth 0.31 to 0.47 bp of each committed close, all of it from LPs. The trader still nets about −7.85 USDG per 10,000-notional round after fees, VPI and bounties. This is not a net-profitable extraction.

Limits a triager will raise, and that we accept:

- **The window is not guaranteed.** The choice must be made before the order's first execution. Close bounties are zero (`SECURITY.md:293-296`), but any caller can execute the close: the trader, the protocol keeper (which is told to "Service the canonical Router head regardless of its reward", `DEPLOYMENT.md:359`), liquidators, or a keeper clearing the head to reach a bountied open. A close may stay pending until its deadline only if no other caller processes it. The tests show a decision at commit + 52 s with nobody executing in between.
- **Atomic exercise is possible, but not guaranteed.** An account that is a batching smart contract could call `addMargin` (or `withdraw`) and `executeOrder` in one transaction. The tests do not demonstrate this; their trader is a plain address. Atomicity prevents interleaving between those two calls. It does not stop another caller from executing the order first. An ordinary EOA's two separate transactions have no such guarantee.
- **Capital.** Lever 1's keep branch parks the trader's own free settlement as margin until the residual closes. It stays the trader's.

**The promise, and the documented policy next to it.**

- The promise: `SECURITY.md:658-659`: "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options". `SECURITY.md:719`: "users cannot cancel a queued intent". `SECURITY.md:1345`: "users cannot manually cancel queued intents".
- The intended failure policy: `SECURITY.md:726-729`: "exact-shape typed planner rejections, execution-mode violations, and financial-constraint violations fail terminally", and "ordinary close failure settles its zero Order record and pays no executor".
- The intended bounds: `SECURITY.md:756-763` and `CLOSE_PREVIEW.md:88-90` ("Execution independently requires a healthy residual position and collectible partial-close charges").
- The whitepaper's softer wording: `WHITEPAPER.md:1137-1138`: "Binding orders reduce the ability to treat a queue position as a free option".

Each ingredient is intended. We found no statement that accepts their combination as a trader-controlled choice after commit, and `SECURITY.md:658-659` states the opposite. The no-cancel statements (`:719`, `:1345`) remain literally true of the API: inducing a terminal failure is not calling a cancel function.

## Relation to prior findings

| Item | What it says | How this report relates |
|---|---|---|
| Pre-audit **SC-M4** "Risk Free Option in 60Sec Oracle Window" (SC Audit Studio, fixed: "Fixed in Plether-Fi/plether-core#24") | "a SOLVENCY_EXCEEDED→RefundUser margin-return path triggered by LP withdrawal … an attacker can effectively hold a costless directional option on the underlying price for up to one minute per order pair" | **Same class, later levers: the fix is incomplete.** At `a82d4fef` orders are binding. There is no user cancellation, the fill is the unique first tick after commit (`SECURITY.md:647-649`), and the `RefundUser` path is gone. The protocol states that this prevents free options (`SECURITY.md:658-659`). The two levers here came later. The equity floor first appears in the repository history on 2026-08-25, after PRs #66 and #70 had merged, so after #24. The `PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE` rejection first appears on 2026-08-21. The maintainers closed the matching settlement-floor lever: a positive settlement floor on a close is rejected at commit (`OrderRouter__SettlementFloorUnsupported`, `OrderLiquidationBatchLogic.sol:308-310`; `CLOSE_PREVIEW.md:88`). The equity floor and the charge collection got no such guard. The #24 diff itself is not in the competition repository, so this framing rests on the fix's stated effect, not on a reading of the patch. |
| Cecuro **HIGH-1** (Info, open known issue: Arbitrum/Pyth clock alignment) | A trader may commit after a Pyth price is already published and be filled at it. A new report on this root needs "a repeatable, net-profitable extraction from LPs beyond this limitation". | **Different root, and that bar is not met.** HIGH-1 concerns information at commit. This report needs no clock lag. Our repeated-trade test measures the trader net-negative after costs, so we do not claim HIGH-1's bar. |
| Cecuro **MED-16** (open) | "Price-dependent checks remain freshness-gated and are revalidated at execution; this does not guarantee that every admitted order will fill." | Different. MED-16 covers opens admitted against a stale cached mark that may fail. Here the trader controls whether the order fails, after commit. |
| `SECURITY.md:726-729`, `CLOSE_PREVIEW.md:88-90` | Typed planner rejections and financial-constraint violations are terminal. Execution independently requires a healthy residual and collectible partial-close charges. | These document *when* the checks run and that failures are terminal. Neither says the trader may decide the outcome after commit, and `SECURITY.md:658-659` says the trader cannot. |
| Cecuro **LOW-7**, AI scan `addMargin` carry-checkpoint item | `addMargin(account, 0)` error classification; one-atom `addMargin` carry checkpointing | Unrelated uses of `addMargin`. |

The collaborative (Sherlock) and Cantina reports cover the spot package and have no related item.

## Vulnerability details

### Lever 1: the equity floor is not read at commit, is terminal at execution, and `addMargin` moves it

Commit-time validation reads the settlement floor but never the equity floor:

```solidity
// OrderLiquidationBatchLogic.sol:303-312 (_validateFreshRequest)
uint64 head = host.accountHeadOrderId(account);
if (head != 0 && host.lifecycleBook().pendingPolicy(head).minPostSettlementBalanceAssets != 0) {
    revert OrderRouter__SettlementFloorOrderPending(head);
}
if (bounds.minPostSettlementBalanceAssets != 0) {
    (uint256 positionSize,,,,,,) = host.engine().positions(account);
    if (request.isClose || positionSize != 0 || head != 0) {
        revert OrderRouter__SettlementFloorUnsupported();
    }
}
```

At execution, a partial close is checked against the floor at the fill price:

```solidity
// CfdOrderPolicyEvaluatorBase.sol:530-539
uint256 normalizedEquityAssets =
    assessment.postPositionEquityAssets > 0 ? uint256(assessment.postPositionEquityAssets) : 0;
if (assessment.postPositionEquityAssets < 0 || normalizedEquityAssets < bounds.minPostPositionEquityAssets) {
    revert ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation(
        OrderV3Types.ConstraintKind.PostPositionEquity,
        normalizedEquityAssets,
        bounds.minPostPositionEquityAssets
    );
}
_requireMaximum(OrderV3Types.ConstraintKind.PostLeverage, assessment.postLeverageBps, bounds.maxPostLeverageBps);
```

That revert is terminal:

```solidity
// OrderRouterExecutionSidecar.sol:879-891
if (
    selector == ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector
        && revertData.length == 100
) {
    uint256 constraint = _word(revertData, 4);
    if (constraint > 0 && constraint <= uint256(OrderV3Types.ConstraintKind.PostLeverage)) {
        classification.terminal = true;
        classification.reason = OrderV3Types.TerminalReason.ConstraintViolation;
        // ...
    }
}
```

The owner can raise the equity the floor reads at any time. There is no pending-order check and no oracle input:

```solidity
// CfdEngine.sol:501-527 (abridged)
function addMargin(address account, uint256 amount) external nonReentrant {
    if (msg.sender != account) revert CfdEngine__NotAccountOwner();
    if (amount == 0) revert CfdEngine__ZeroAmount();
    StoredPosition storage pos = _positions[account];
    if (pos.lots == 0) revert CfdEngine__NoOpenPosition();
    // ... realize carry, then:
    clearinghouse.lockPositionMargin(account, amount);
    _sideState(pos.side).totalMargin += amount;
    // ...
}
```

Full closes skip the residual checks (`CfdOrderPolicyEvaluatorBase.sol:526-528`), so lever 1 needs a surviving position. `maxPostLeverageBps` is checked at the same point and also moves with `addMargin`. We did not test it separately.

**The same lever on an increase.** When the trader executes its own open and it fails on a typed check, the reserved bounty goes back to the trader's account (`OrderRouter.sol:336-337`, context `:333-342`). An increase committed with an unreachable `minPostPositionEquityAssets` therefore fails without a USDG debit when the trader executes it itself. If a third party executes it first, the failed branch pays that executor the bounty.

### Lever 2: an ordinary close's charge collection reads free settlement, which the owner can withdraw after commit

A partial close collects its action charge (fee, VPI, carry) first from the spendable action reserve, then from free settlement, then from released margin limited by health (`CfdEnginePlanLib._planCloseCashSources`, `CfdEnginePlanLib.sol:1092-1133`; `_limitCloseReleaseByHealth`, `:1137-1180`). On a partial close, any charge it cannot collect is a terminal rejection:

```solidity
// CfdEnginePlanLib.sol:882-885
if (remainingSize > 0 && delta.actionChargeWaivedAssets > 0) {
    delta.revertCode = CfdEnginePlanTypes.CloseRevertCode.PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE;
    return delta;
}
```

The Router records that as `PlannerRejected` and the order fails. Free settlement is not position collateral. The owner can withdraw it while the close is queued, because neither `MarginClearinghouse._withdraw` (`MarginClearinghouse.sol:441-477`) nor `CfdEngine.checkWithdraw` (`CfdEngine.sol:730-748`) reads the order queue. Where health limits the released margin to less than the charge, leaving the free settlement in place lets the close execute, and withdrawing it makes the close fail. The trader needs no unusual bound for this. The band is narrow: in the tests it is about 1.5 to 2 bp of adverse move wide.

## Impact

**Grade: Insight.** We do not claim an impact from the program's list, for these reasons:

- **Not the Low line.** The Low line is "Contract fails to deliver promised returns, but doesn't lose value". What breaks here is a promise about queue semantics, not a promise of financial returns. Where the choice is exercised with later price information, it moves LP value to the trader, which is a loss of value, however small.
- **No value-loss line is demonstrated.** Over repeated trades at the release parameters and the shipped 60 s window, the information-conditioned choice is worth 0.31 to 0.47 bp of each committed close. Under an assumed 7.5% annual volatility that is about 1.6 to 2.4 USDG per 50,000 closed, all of it from LPs (USDG taken at USDC parity). After fees, VPI and bounties the trader nets about −7.85 USDG per 10,000-notional round. This is not theft and not a net-profitable extraction.
- **Not the Medium griefing line.** A trader running the lever only to hurt LPs pays about 30 times what LPs lose: about 7.75 USDG of fees and bounty per round against about 0.24 USDG of option value.

What LPs give up is the option value on closes that traders make anyway, each time a trader uses the lever with a price signal. It grows with notional and volatility, and roughly with the square root of the time between the fill tick and the decision. All value figures above are frictionless and gross except the measured trader net. The 1 h owner cap on the window (`OrderRouterAdmin.sol:31`, behind a 72 h timelock) gives about 3.2 bp per close by `√t` scaling on a 365.25-day year (about 3.8 bp on a 252-trading-day year). That figure was not executed.

Why report it at all: the protocol states a guarantee that its own code does not provide, and the maintainers already closed the identical settlement-floor lever. Either the two remaining levers should be closed the same way, or the guarantee should be restated as the whitepaper's "reduce".


## Recommendation

Close the two remaining levers the way the settlement floor was closed, or restate the guarantee. These are sketches. We have not implemented them or run them against the maintainers' suite.

1. **Lever 1, closes.** In `_validateFreshRequest`, reject a nonzero `minPostPositionEquityAssets`, and a `maxPostLeverageBps` below the type maximum, on closes, next to the existing `OrderRouter__SettlementFloorUnsupported` check. This removes the demonstrated lever. It does not necessarily remove every way an owner action can change whether a close passes at execution.
2. **Lever 1, increases.** Checking the bound once at commit does not fix the outcome at execution, because later owner actions still change the inputs. Either reject the bound on increases too, or make the inputs the bound reads fixed for the order's lifetime: reserve the funding at commit, as the open-side settlement floor already does (`OrderLiquidationBatchLogic.sol:259-266`).
3. **Lever 2.** Reserve a queued partial close's maximum action charge at commit, as an open's bounty is reserved, or refuse withdrawals that would take free settlement below that reservation while the close is queued. A withdrawal restriction is safer than blocking `addMargin`: the existing `OrderRouter__SettlementFloorOrderPending` guard (`OrderLiquidationBatchLogic.sol:303-306`) blocks *admitting a new order* behind a pending settlement-floor order. It is not an `addMargin` lock. Blocking margin top-ups while an order is pending could stop a trader from rescuing a position near liquidation, so we do not recommend that.
4. **If the behaviour is intended,** change `SECURITY.md:658-659` to the whitepaper's wording ("reduce the ability to treat a queue position as a free option"), and list both levers as accepted.

We do not recommend making these violations retryable instead of terminal. The order would stay pending, and a zero-bounty close that the trader does not top up would still end at its deadline. Expiry is terminal too (`SECURITY.md:724`; `OrderRouterExecutionSidecar.sol:910-911`), so the choice would remain.

The property tests do not become regression tests unchanged. Under fix 1, a close with a nonzero equity floor reverts at commit, so the lever 1 property should assert that admission rejection. Under fix 3, the lever 2 property's withdrawal should revert or leave the close executable.


## Link to Proof of Concept

https://gist.github.com/Richter2552/f10111e922c7bef6b3790e193e6284fe

## Proof of Concept

## Proof of Concept

The program requires PoCs to run in the repository's Foundry test suite at the commit in scope without modifying the contracts in scope. The three test files below extend the maintainers' `BasePerpTest` fixture. Prices come from the fixture's `MockPyth`; the program allows mocking Pyth. Every action after setup is a public call:

- the trader's own `router.commitOrder`, `engine.addMargin`, `clearinghouse.withdraw` and (for the increase) `router.executeOrder`;
- a keeper with no role calling `router.executeOrder`.

Setup positions are opened through the Router commit and a keeper execution. No protocol contract is impersonated, and no `vm.store` or `vm.etch` is used.

**When this insight is valuable.** All of the following must hold:

1. The trader can make its decision before anyone else executes the queued close.
2. The trader has a price signal after commit, such as the published fill tick and a later market price.
3. The account holds the free settlement that the lever needs: the top-up for lever 1, or the withdrawable balance for lever 2.
4. For lever 2, the fill falls in the narrow band where health-limited released margin is below the action charge.

Then the trader keeps the close only when the fill beats the later price. The gain is roughly `σ·√t/√(2π)` of the closed notional, paid by LPs. Fees exceed this on any trade made only to harvest it. So it matters for traders who are closing anyway, and it grows with a longer owner-set window or a more volatile basket.

Files:

- `Card6482S1CloseFreeOption.t.sol` (lever 1). Setup: TRADER holds a 10,000-token LONG with 2,000 USDG margin and commits a partial close of 5,000 with `minPostPositionEquityAssets = 3,000 USDG`; on its own the residual's equity is about 1,000. The two arms share the state and the fill tick, and differ only in whether TRADER calls `addMargin(5,000)` before execution.
- `Card5394S1WithdrawLever.t.sol` (lever 2), at two parameter sets:
  - the fixture's own parameters (TRADER 10,000 LONG with 200 USDG margin);
  - the release parameters from `DEPLOYMENT.md` (TRADER 10,000 LONG with 50 USDG margin, with BOB 30,000 SHORT so that TRADER's close pays VPI rather than earning a rebate).

  The close is ordinary, with no equity floor. The arms differ only in whether TRADER withdraws all its free settlement after commit.
- `Card6482S1RepeatedTrade.t.sol` measures what lever 1's choice is worth over many trades ("Repeated-trade measurement" below).

The **property** tests state the behaviour the code should have and **fail** on the pinned code. The **control** and **characterize** tests pass and pin the exact behaviour.

Run instructions (from a checkout of `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with submodules; solc 0.8.35 via the repository's `packages/perps/foundry.toml`):

```bash
cp Card6482S1CloseFreeOption.t.sol Card5394S1WithdrawLever.t.sol Card6482S1RepeatedTrade.t.sol \
  packages/perps/test/perps/spec/trader/
cd packages/perps

forge test --match-contract '^Card6482S1CloseFreeOptionTest$' --match-test '^test_Property_' -vv
forge test --match-contract '^Card5394S1WithdrawLever(Release)?Test$' --match-test '^test_Property_' -vv

forge test --match-contract '^Card6482S1CloseFreeOptionTest$' --match-test '^test_(Control|Characterize)_' -vv
forge test --match-contract '^Card5394S1WithdrawLever(Release)?Test$' --match-test '^test_(Control|Characterize)_' -vv

forge test --match-contract '^Card6482S1RepeatedTradeTest$' --match-test '^test_Measure_' -vv
```

### PoC output, lever 1 (pinned code, unmodified contracts)

Run against the pinned commit (forge 1.8.1, solc 0.8.35, via-IR, the repository's remappings): 3 tests pass and the 2 property tests fail. Excerpt (forge `-vv` logs; status 2 = Executed, 3 = Failed; terminal reason 8 = `ConstraintViolation`; custody is `clearinghouse.balanceAssets` in 1e6 units; sizes in 1e18; the keeper's 200000 is the bounty it earned on the setup open):

```console
[PASS] test_Characterize_BothOutcomesAtGoodAndBadFills() (gas: 24737603)
Logs:
  fill +2% DROP
    status (2=Executed,3=Failed) 3
    terminalReason 8
    trader size after 10000000000000000000000
    trader custody before/after 19995799999 19995799999
    keeper custody before/after 200000 200000
  fill +2% KEEP
    status (2=Executed,3=Failed) 2
    terminalReason 1
    trader size after 5000000000000000000000
    trader custody before/after 19995799999 19893759999
    keeper custody before/after 200000 200000
  fill -2% DROP
    status (2=Executed,3=Failed) 3
    terminalReason 8
    trader size after 10000000000000000000000
    trader custody before/after 19995799999 19995799999
    keeper custody before/after 200000 200000
  fill -2% KEEP
    status (2=Executed,3=Failed) 2
    terminalReason 1
    trader size after 5000000000000000000000
    trader custody before/after 19995799999 20093839999
    keeper custody before/after 200000 200000

[PASS] test_Characterize_LateDecisionStillFillsAtOriginalTick() (gas: 12589277)
Logs:
  max execution window (s) 60
  decision made at commit + (s) 52
  executed at commit + (s) 53
  fill tick at commit + (s) 1
  commitTime / deadline 1709535611 1709535671
  LATE DROP
    status (2=Executed,3=Failed) 3
    terminalReason 8
    trader size after 10000000000000000000000
    trader custody before/after 19995799999 19995799999
    keeper custody before/after 200000 200000
  LATE KEEP
    status (2=Executed,3=Failed) 2
    terminalReason 1
    trader size after 5000000000000000000000
    trader custody before/after 19995799999 20093839994
    keeper custody before/after 200000 200000

[PASS] test_Control_NoEquityBound_ExecutesEitherWay() (gas: 12575773)
Suite result: ok. 3 passed; 0 failed; 0 skipped

[FAIL: free option: trader picks execute-or-drop after the fill tick is known: 3 != 2] test_Property_TraderCannotChooseOutcomeAfterFill() (gas: 13027043)
Logs:
  DROP (trader does nothing)
    status (2=Executed,3=Failed) 3
    terminalReason 8
    trader size after 10000000000000000000000
    trader custody before/after 19995799999 19995799999
    keeper custody before/after 200000 200000
  KEEP (trader addMargin first)
    status (2=Executed,3=Failed) 2
    terminalReason 1
    trader size after 5000000000000000000000
    trader custody before/after 19995799999 19993799999
    keeper custody before/after 200000 200000

[FAIL: free option on an increase: trader picks after the fill tick: 3 != 2] test_Property_IncreaseSelfExecutedIsAlsoAFreeOption() (gas: 13860638)
Logs:
  INCREASE DROP (self-executed, no top-up)
    status (2=Executed,3=Failed) 3
    terminalReason 8
    trader size after 10000000000000000000000
    trader custody before/after 19995799999 19995799999
    keeper custody before/after 200000 200000
  INCREASE KEEP (addMargin, self-executed)
    status (2=Executed,3=Failed) 2
    terminalReason 1
    trader size after 15000000000000000000000
    trader custody before/after 19995799999 19993799999
    keeper custody before/after 200000 200000
```

### PoC output, lever 2

Excerpt from the same run; the two `#` lines are our labels, not forge output. Terminal reason 7 = `PlannerRejected`; the planner's `previewClose` reason 7 = `PartialActionChargeUncollectible`. Fills are in 0.1 bp of adverse move (a LONG loses when the mark rises).

```console
[PASS] test_Characterize_SweepBand() (gas: 780417607)
Logs:
  diverges at adverse fill (0.1 bp) 1335
    keep status/reason/previewReason 2 1 0
    kill status/reason/previewReason 3 7 7
  diverges at adverse fill (0.1 bp) 1340
    keep status/reason/previewReason 2 1 0
    kill status/reason/previewReason 3 7 7
  diverges at adverse fill (0.1 bp) 1345
    keep status/reason/previewReason 2 1 0
    kill status/reason/previewReason 3 7 7
  diverges at adverse fill (0.1 bp) 1350
    keep status/reason/previewReason 2 1 0
    kill status/reason/previewReason 3 7 7
  sweep from/to (0.1 bp) 1200 1500
  diverging fills (0.5 bp grid) 4
  band first/last (0.1 bp) 1335 1350
[PASS] test_Control_BelowBandWithdrawalIsInert() (gas: 13303096)
[PASS] test_Control_FlatFillExecutesEitherWay() (gas: 13060695)
[FAIL: free option: an ordinary withdrawal after commit decides the queued close: 3 != 2] test_Property_WithdrawAfterCommitCannotKillQueuedPartialClose() (gas: 13604690)
Logs:
  adverse fill (0.1 bp) 1350
  KEEP (trader does nothing)
    status (2=Executed,3=Failed) / terminalReason 2 1
    trader size after 5000000000000000000000
  KILL (trader withdraws its free settlement first)
    status (2=Executed,3=Failed) / terminalReason 3 7
    planner previewClose invalidReason 7
    trader size after 10000000000000000000000
    custody before / withdrawn / custody after 1995799999 1799800000 195999999
    keeper custody before/after 200000 200000

[PASS] test_Characterize_SweepBand() (gas: 600306739)
Logs:
  diverges at adverse fill (0.1 bp) 315
    keep status/reason/previewReason 2 1 0
    kill status/reason/previewReason 3 7 7
  diverges at adverse fill (0.1 bp) 320
    keep status/reason/previewReason 2 1 0
    kill status/reason/previewReason 3 7 7
  diverges at adverse fill (0.1 bp) 325
    keep status/reason/previewReason 2 1 0
    kill status/reason/previewReason 3 7 7
  diverges at adverse fill (0.1 bp) 330
    keep status/reason/previewReason 2 1 0
    kill status/reason/previewReason 3 7 7
  sweep from/to (0.1 bp) 250 400
  diverging fills (0.5 bp grid) 4
  band first/last (0.1 bp) 315 330
[PASS] test_Control_BelowBandWithdrawalIsInert() (gas: 19798398)
[PASS] test_Control_FlatFillExecutesEitherWay() (gas: 19726091)
[FAIL: free option: an ordinary withdrawal after commit decides the queued close: 3 != 2] test_Property_WithdrawAfterCommitCannotKillQueuedPartialClose() (gas: 20552408)
Logs:
  adverse fill (0.1 bp) 320
  KEEP (trader does nothing)
    status (2=Executed,3=Failed) / terminalReason 2 1
    trader size after 5000000000000000000000
  KILL (trader withdraws its free settlement first)
    status (2=Executed,3=Failed) / terminalReason 3 7
    planner previewClose invalidReason 7
    trader size after 10000000000000000000000
    custody before / withdrawn / custody after 1998294997 1947305002 50989995
    keeper custody before/after 400000 400000
```

### What the tests show

- **Lever 1 property (fails):** same order, same fill tick, identical custody when the decision is made. If the trader does nothing, the order ends `Failed` (`ConstraintViolation`): no close is settled, the position size is unchanged, custody is unchanged and the keeper is paid nothing. If the trader calls `addMargin` first, the order ends `Executed` and the position halves.
- **Custody is realised PnL, not the value of the choice.** A LONG profits when the basket mark falls (`CfdTypes.sol:14-17`). Keeping the close at a −2% fill credits +98.04 USDG, and at a +2% fill it debits −102.04 USDG. Both are realised PnL on the closed half against the open price. Dropping the close at a bad fill does not remove the open position's loss, which stays in the unclosed half. The choice is worth the gap between the fill and the price at the decision. For a LONG close that is `P_decision − P_fill` when positive. The single-trade tests do not model a decision-time price; the repeated-trade test does.
- **Late decision, two clocks:** the decision (`addMargin` or nothing) is at commit + 52 s, execution at commit + 53 s (the fixture's oracle helper advances one second), and the fill tick at commit + 1 s. So the decision comes 51 s after the fill tick, inside the shipped 60 s window. Keeping still fills at the original tick: +98.039995 USDG, five atoms less than the immediate case, consistent with carry over the longer wait. Dropping still debits nothing and pays no executor. The 15 s historical search limit (`PletherOracle.sol:86-87`) bounds where the fill tick may sit after commit. It does not bound when the trader decides.
- **Lever 1 on an increase (fails):** a 5,000 increase with `minPostPositionEquityAssets = 6,000`, executed by the trader itself. Without a top-up it fails with `ConstraintViolation` and custody is exactly unchanged, because the bounty returns to the trader. With `addMargin` first, it executes.
- **Lever 2 property (fails) at both parameter sets:** same order and fill, with the same custody, free settlement and mark in both arms. Leaving free settlement in place, the close executes. Withdrawing it first, the close fails as `PlannerRejected`, the planner's own reason is `PartialActionChargeUncollectible`, and custody left plus the amount withdrawn equals custody before. The keeper is paid nothing.
- **Lever 2 band:** the arms diverge only between +1.335% and +1.350% adverse at the fixture parameters, and between +0.315% and +0.330% at the release parameters (0.5 bp grid). Below the band the withdrawal changes nothing (control). Above it, both arms fail as unhealthy.
- **Controls:** lever 1 with the floor at 0, and lever 2 at an unchanged fill, execute in both arms. The effect comes from the levers, not from the fixture.

### Test file: `packages/perps/test/perps/spec/trader/Card6482S1CloseFreeOption.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

/// @title A queued partial close (or increase) can be kept or dropped by its trader after its fill tick is known
/// @notice Plether audit competition, immunefi-team/audit-comp-plether @ a82d4fefb02768e5b3bddcc97dee371eee24fea8.
/// @dev PROMISE (packages/perps/SECURITY.md:658-659, "Delayed-order model"):
///        "binding order semantics and the lack of a user cancellation path prevent traders from turning queued
///         intents into free options"
///      and SECURITY.md:719 "users cannot cancel a queued intent".
///
///      One self-cancellation lever on closes is already closed: a positive settlement floor on a close is rejected
///      at commit (CLOSE_PREVIEW.md:88, `OrderRouter__SettlementFloorUnsupported`). A second bound has the same shape
///      and is not closed:
///
///        - `ExecutionBounds.minPostPositionEquityAssets` is accepted on a partial close and is not checked at commit.
///        - At execution it is enforced against the residual position's equity at the fill price
///          (CfdOrderPolicyEvaluatorBase.sol:530-538). A violation is TERMINAL (`ConstraintViolation`,
///          OrderRouterExecutionSidecar.sol:879-891), and a failed close pays no executor and debits the trader
///          nothing (SECURITY.md:728-729 "ordinary close failure settles its zero Order record and pays no executor").
///        - `CfdEngine.addMargin` (CfdEngine.sol:501-527) lets the owner move free settlement into the position's
///          pledge at any time: no pending-order check, no oracle input. It raises the residual equity the bound reads.
///
///      So a trader commits a partial close whose equity bound is set above what the residual can reach. After
///      commit, the trader decides:
///        - do nothing  -> the next execution fails it terminally: no close is settled, the position size is unchanged,
///                         and the attempt debits the trader no USDG and pays no executor (commitment's own gas and
///                         carry, and the open position's market risk and carry, remain);
///        - addMargin   -> the same execution, at the same fill tick, succeeds.
///
///      What these tests prove is the SWITCH: at a fixed historical fill, the outcome depends on a post-commit owner
///      action. The mocked historical tick is constructed at execution, after the trader's action, so the tests do not
///      stage "observe the published fill, observe a later market price, then choose". The economic value of the
///      choice is measured separately in Card6482S1RepeatedTrade.t.sol. The strict status equality in the two
///      property tests is the requirement we propose, not an invariant the code documents in that form.
///
///      Every step is a public call by an account with no role. No in-scope contract is modified, no vm.store/etch.
///      The setup open goes through the Router commit and a keeper execution, not a pranked engine call.
contract Card6482S1CloseFreeOptionTest is BasePerpTest {
    address internal constant TRADER = address(0x7A11CE);
    address internal constant KEEPER = address(0x6EE9E9);

    uint256 internal constant P0 = 1e8; // 1.00
    uint256 internal constant SIZE = 10_000e18;
    uint256 internal constant HALF = 5000e18;
    uint256 internal constant OPEN_MARGIN = 2000e6;
    uint256 internal constant FUNDING = 20_000e6;
    /// Equity floor for the residual 5,000 LONG: above anything the residual reaches on its own (its pledge is ~1,000
    /// and price PnL near P0 is a few USDG), below what it reaches after the trader tops up 5,000.
    uint256 internal constant EQUITY_FLOOR = 3000e6;
    uint256 internal constant TOP_UP = 5000e6;

    struct Close {
        uint64 orderId;
        uint64 commitTime;
    }

    function _size(
        address a
    ) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _execute(
        uint64 orderId,
        uint256 fillPrice
    ) internal returns (OrderV3Types.ExecutionResult memory res) {
        // Unique historical tick at commitTime + 1 for the queue head (BasePerpTest._mockHistoricalPublishTime).
        bytes[] memory hist = _mockPythUpdateData(fillPrice);
        vm.prank(KEEPER);
        res = router.executeOrder(orderId, hist);
    }

    function _openViaRouter() internal {
        _fundTrader(TRADER, FUNDING);
        vm.prank(TRADER);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, SIZE, OPEN_MARGIN, 0, false);
        OrderV3Types.ExecutionResult memory res = _execute(openId, P0);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "setup: open executed");
        assertEq(_size(TRADER), SIZE, "setup: position open");
    }

    /// Partial close of half the LONG with every bound as loose as the Router's own scalar helper, except the
    /// residual-equity floor.
    function _commitPartialClose(
        uint256 minPostPositionEquityAssets
    ) internal returns (Close memory c) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("card6482-close", router.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = HALF;
        request.marginDelta = 0;
        request.targetPrice = type(uint256).max; // a LONG close accepts any price at or below target
        request.isClose = true;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: 0,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: minPostPositionEquityAssets,
            maxPostLeverageBps: type(uint32).max
        });
        vm.warp(vm.getBlockTimestamp() + 10);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(TRADER);
        c.orderId = router.commitOrder(request);
        c.commitTime = _orderRecord(c.orderId).core.commitTime;
    }

    struct Arm {
        uint8 status;
        uint8 terminalReason;
        uint256 sizeAfter;
        uint256 custodyBefore;
        uint256 custodyAfter;
        uint256 keeperBefore;
        uint256 keeperAfter;
    }

    /// One arm: same world, same committed close, same fill tick; `keep` = whether the trader tops up first.
    function _arm(
        uint256 equityFloor,
        uint256 fillPrice,
        bool keep
    ) internal returns (Arm memory a) {
        vm.deal(KEEPER, 10 ether);
        _openViaRouter();
        Close memory c = _commitPartialClose(equityFloor);
        a.custodyBefore = clearinghouse.balanceAssets(TRADER);
        a.keeperBefore = clearinghouse.balanceAssets(KEEPER);
        if (keep) {
            // No oracle input and no queue check on addMargin.
            vm.prank(TRADER);
            engine.addMargin(TRADER, TOP_UP);
        }
        OrderV3Types.ExecutionResult memory res = _execute(c.orderId, fillPrice);
        a.status = uint8(res.status);
        a.terminalReason = uint8(res.terminalReason);
        a.sizeAfter = _size(TRADER);
        a.custodyAfter = clearinghouse.balanceAssets(TRADER);
        a.keeperAfter = clearinghouse.balanceAssets(KEEPER);
    }

    function _log(
        string memory tag,
        Arm memory a
    ) internal pure {
        console2.log(tag);
        console2.log("  status (2=Executed,3=Failed)", a.status);
        console2.log("  terminalReason", a.terminalReason);
        console2.log("  trader size after", a.sizeAfter);
        console2.log("  trader custody before/after", a.custodyBefore, a.custodyAfter);
        console2.log("  keeper custody before/after", a.keeperBefore, a.keeperAfter);
    }

    /// PROPERTY (FAILS on the pinned code): once a close is committed and its fill tick is fixed, whether it
    /// executes must not depend on a choice the trader makes afterwards.
    function test_Property_TraderCannotChooseOutcomeAfterFill() public {
        uint256 s = vm.snapshotState();
        Arm memory drop = _arm(EQUITY_FLOOR, P0, false);
        vm.revertToState(s);
        Arm memory keep = _arm(EQUITY_FLOOR, P0, true);
        vm.revertToState(s);

        _log("DROP (trader does nothing)", drop);
        _log("KEEP (trader addMargin first)", keep);

        // The two arms differ only in the trader's post-commit action.
        assertEq(drop.custodyBefore, keep.custodyBefore, "same custody at decision time");
        // DROP arm: the failed attempt debits the trader nothing and pays no executor.
        assertEq(drop.status, uint8(OrderV3Types.LifecycleStatus.Failed), "DROP fails terminally");
        assertEq(drop.sizeAfter, SIZE, "DROP leaves the position size unchanged");
        assertEq(drop.custodyAfter, drop.custodyBefore, "DROP debits the trader nothing");
        assertEq(drop.keeperAfter, drop.keeperBefore, "DROP pays no executor");
        // KEEP arm executes at the same fill.
        assertEq(keep.status, uint8(OrderV3Types.LifecycleStatus.Executed), "KEEP executes");
        assertEq(keep.sizeAfter, SIZE - HALF, "KEEP halves the position");

        assertEq(drop.status, keep.status, "free option: trader picks execute-or-drop after the fill tick is known");
    }

    /// Both outcomes at both fills: a LONG profits when the basket mark falls (CfdTypes.sol:14-17). The custody change
    /// on a kept close is the realised PnL of the closed half against the open price; it is not the value of the
    /// choice. The choice is worth the gap between the fill and the price at the trader's decision (for a LONG close,
    /// P_decision - P_fill when positive), which this test does not model; see Card6482S1RepeatedTrade.t.sol.
    function test_Characterize_BothOutcomesAtGoodAndBadFills() public {
        uint256 s = vm.snapshotState();
        Arm memory upDrop = _arm(EQUITY_FLOOR, P0 * 102 / 100, false);
        vm.revertToState(s);
        Arm memory upKeep = _arm(EQUITY_FLOOR, P0 * 102 / 100, true);
        vm.revertToState(s);
        Arm memory downDrop = _arm(EQUITY_FLOOR, P0 * 98 / 100, false);
        vm.revertToState(s);
        Arm memory downKeep = _arm(EQUITY_FLOOR, P0 * 98 / 100, true);
        vm.revertToState(s);
        _log("fill +2% DROP", upDrop);
        _log("fill +2% KEEP", upKeep);
        _log("fill -2% DROP", downDrop);
        _log("fill -2% KEEP", downKeep);
        assertEq(upDrop.status, uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(upKeep.status, uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(downDrop.status, uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(downKeep.status, uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    // ------------------------------------------------------------------ the decision window on a CLOSE
    // Close bounties are zero by design (SECURITY.md:293-296) and execution is head-only on one FIFO, so no order
    // bounty pays anyone to run a lone queued close. It can still be executed by any caller (the trader, a protocol
    // keeper, a liquidator, a keeper clearing the head to reach a bountied open). If nobody does, the trader can decide
    // late in the execution window, and the close still fills at the ORIGINAL historical tick (commitTime + 1).

    function _lateArm(
        bool keep,
        uint256 waitSeconds
    ) internal returns (Arm memory a, uint64 commitTime, uint64 deadline, uint256 executedAt) {
        vm.deal(KEEPER, 10 ether);
        _openViaRouter();
        Close memory c = _commitPartialClose(EQUITY_FLOOR);
        commitTime = c.commitTime;
        deadline = _orderRecord(c.orderId).core.commitTime + uint64(router.maxExecutionWindowSeconds());
        a.custodyBefore = clearinghouse.balanceAssets(TRADER);
        a.keeperBefore = clearinghouse.balanceAssets(KEEPER);
        // Nobody executes the zero-bounty close in the meantime.
        vm.warp(uint256(c.commitTime) + waitSeconds);
        vm.roll(vm.getBlockNumber() + 1);
        vm.startPrank(TRADER);
        if (keep) {
            engine.addMargin(TRADER, TOP_UP);
        }
        vm.stopPrank();
        // _mockPythUpdateData advances the clock one more second (BasePerpTest.sol:304-307) and publishes the head
        // order's unique historical tick at commitTime + 1. So the decision is at commit + waitSeconds, execution at
        // commit + waitSeconds + 1, and the fill tick is at commit + 1.
        OrderV3Types.ExecutionResult memory res = _execute(c.orderId, P0 * 98 / 100);
        executedAt = vm.getBlockTimestamp();
        a.status = uint8(res.status);
        a.terminalReason = uint8(res.terminalReason);
        a.sizeAfter = _size(TRADER);
        a.custodyAfter = clearinghouse.balanceAssets(TRADER);
        a.keeperAfter = clearinghouse.balanceAssets(KEEPER);
    }

    /// Characterization: with the close alone at the head, the trader's choice can be made late in the window and the
    /// close still fills at the original tick. Logs the window the fixture allows.
    function test_Characterize_LateDecisionStillFillsAtOriginalTick() public {
        uint256 window = router.maxExecutionWindowSeconds();
        uint256 waitSeconds = window > 10 ? window - 8 : 1;
        uint256 s = vm.snapshotState();
        (Arm memory drop, uint64 commitTime, uint64 deadline, uint256 executedAt) = _lateArm(false, waitSeconds);
        vm.revertToState(s);
        (Arm memory keep,,,) = _lateArm(true, waitSeconds);
        vm.revertToState(s);
        console2.log("max execution window (s)", window);
        console2.log("decision made at commit + (s)", waitSeconds);
        console2.log("executed at commit + (s)", executedAt - commitTime);
        console2.log("fill tick at commit + (s)", uint256(1));
        console2.log("commitTime / deadline", commitTime, deadline);
        _log("LATE DROP", drop);
        _log("LATE KEEP", keep);
        assertEq(drop.status, uint8(OrderV3Types.LifecycleStatus.Failed), "late drop fails terminally");
        assertEq(drop.custodyAfter, drop.custodyBefore, "late drop debits the trader nothing");
        assertEq(drop.keeperAfter, drop.keeperBefore, "late drop pays no executor");
        assertEq(keep.status, uint8(OrderV3Types.LifecycleStatus.Executed), "late keep still fills at the old tick");
        assertEq(keep.sizeAfter, SIZE - HALF, "late keep halves the position");
    }

    // ------------------------------------------------------------------ sibling: an INCREASE (open on a position)
    // A self-executed open that fails on a typed check returns its own bounty to the account (OrderRouter.sol:336-337).
    // With the same addMargin lever, a failed increase debits nothing when the trader executes it itself. If a third
    // party executes it first, the failed branch pays that executor the bounty instead.

    function _commitIncrease(
        uint256 minPostPositionEquityAssets
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("card6482-increase", router.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = HALF;
        request.marginDelta = 1000e6;
        request.targetPrice = 1; // a LONG open accepts any price at or above target
        request.isClose = false;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
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
            minPostPositionEquityAssets: minPostPositionEquityAssets,
            maxPostLeverageBps: type(uint32).max
        });
        vm.warp(vm.getBlockTimestamp() + 10);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(TRADER);
        orderId = router.commitOrder(request);
    }

    function _incArm(
        bool keep
    ) internal returns (Arm memory a) {
        vm.deal(KEEPER, 10 ether);
        _openViaRouter();
        // Post-increase equity is ~2,000 pledge + 1,000 new margin - costs; the floor needs a top-up to pass.
        uint64 id = _commitIncrease(6000e6);
        a.custodyBefore = clearinghouse.balanceAssets(TRADER);
        a.keeperBefore = clearinghouse.balanceAssets(KEEPER);
        bytes[] memory hist = _mockPythUpdateData(P0);
        vm.startPrank(TRADER);
        if (keep) {
            engine.addMargin(TRADER, TOP_UP);
        }
        // The trader executes its own order: on a typed failure the bounty goes back to the trader.
        OrderV3Types.ExecutionResult memory res = router.executeOrder(id, hist);
        vm.stopPrank();
        a.status = uint8(res.status);
        a.terminalReason = uint8(res.terminalReason);
        a.sizeAfter = _size(TRADER);
        a.custodyAfter = clearinghouse.balanceAssets(TRADER);
        a.keeperAfter = clearinghouse.balanceAssets(KEEPER);
    }

    /// PROPERTY (FAILS on the pinned code): a committed increase's outcome must not be the trader's post-commit choice.
    function test_Property_IncreaseSelfExecutedIsAlsoAFreeOption() public {
        uint256 s = vm.snapshotState();
        Arm memory drop = _incArm(false);
        vm.revertToState(s);
        Arm memory keep = _incArm(true);
        vm.revertToState(s);
        _log("INCREASE DROP (self-executed, no top-up)", drop);
        _log("INCREASE KEEP (addMargin, self-executed)", keep);
        assertEq(drop.status, uint8(OrderV3Types.LifecycleStatus.Failed), "drop fails");
        assertEq(drop.terminalReason, uint8(OrderV3Types.TerminalReason.ConstraintViolation), "bound");
        assertEq(drop.sizeAfter, SIZE, "drop leaves the position");
        assertEq(drop.custodyAfter, drop.custodyBefore, "declined increase debits the self-executing trader 0");
        assertEq(keep.status, uint8(OrderV3Types.LifecycleStatus.Executed), "keep executes");
        assertEq(keep.sizeAfter, SIZE + HALF, "keep increases");
        assertEq(drop.status, keep.status, "free option on an increase: trader picks after the fill tick");
    }

    /// CONTROL: without the equity bound the same close executes in both arms, so the lever is the bound plus
    /// addMargin, not the harness.
    function test_Control_NoEquityBound_ExecutesEitherWay() public {
        uint256 s = vm.snapshotState();
        Arm memory a = _arm(0, P0, false);
        vm.revertToState(s);
        Arm memory b = _arm(0, P0, true);
        vm.revertToState(s);
        assertEq(a.status, uint8(OrderV3Types.LifecycleStatus.Executed), "control: executes without top-up");
        assertEq(b.status, uint8(OrderV3Types.LifecycleStatus.Executed), "control: executes with top-up");
    }
}
```

### Test file: `packages/perps/test/perps/spec/trader/Card5394S1WithdrawLever.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @title Second lever on the same promise: an ORDINARY queued partial close (no equity floor) is decided after its
///        fill tick by an ordinary withdrawal of free settlement
/// @notice Plether audit competition, immunefi-team/audit-comp-plether @ a82d4fefb02768e5b3bddcc97dee371eee24fea8.
/// @dev PROMISE (packages/perps/SECURITY.md:658-659, "Delayed-order model"): "binding order semantics and the lack of a
///      user cancellation path prevent traders from turning queued intents into free options".
///
///      A partial close's action charge is collected from the spendable action reserve, then from FREE settlement,
///      then from health-limited released margin (CfdEnginePlanLib._planCloseCashSources :1092-1133,
///      _limitCloseReleaseByHealth :1137-1180). A partial close whose charge cannot be fully collected is a terminal
///      PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE planner rejection (:882-885). Free settlement is not position collateral, so
///      the owner can withdraw it while the close is queued (MarginClearinghouse._withdraw :441-477,
///      CfdEngine.checkWithdraw :730-748; neither reads the order queue). Where health limits the released margin below
///      the charge, the trader decides after commit whether the queued close executes (leave free settlement) or fails
///      terminally with no custody debit and no executor payment (withdraw it).
///
///      The lever needs no unusual bound: the close carries no equity floor. It exists only in a narrow band of fills
///      (a losing close whose residual sits near the release-health limit at the fill), which the sweeps locate.
///
///      As in the companion file, the mocked historical tick is constructed at execution, after the trader's action. The
///      test fixes the fill and shows that the outcome depends on the post-commit action; it does not stage the
///      trader observing a published tick and then a later market price.
///
///      Two contracts:
///        - Card5394S1WithdrawLeverTest: the maintainers' BasePerpTest risk parameters (vpiFactor 0, maint/init
///          100/150 bps). Band near +1.35% adverse.
///        - Card5394S1WithdrawLeverReleaseTest: the release values from packages/perps/DEPLOYMENT.md (vpiFactor 0.01e18,
///          maint/init 10/20 bps; every other field equal). Band near +0.32% adverse.
///
///      Every step is a public call by an account with no role. No in-scope contract is modified, no vm.store/etch.
contract Card5394S1WithdrawLeverTest is BasePerpTest {
    address internal constant TRADER = address(0x7A11CE);
    address internal constant KEEPER = address(0x6EE9E9);

    uint256 internal constant P0 = 1e8; // 1.00
    uint256 internal constant SIZE = 10_000e18;
    uint256 internal constant HALF = 5000e18;
    uint256 internal constant OPEN_MARGIN = 200e6; // ~2% on 10k notional: IM is 1.5%, maintenance 1% in the fixture
    uint256 internal constant FUNDING = 2000e6;

    struct Arm {
        uint8 status;
        uint8 terminalReason;
        uint256 sizeAfter;
        uint256 freeBeforeDecision;
        uint256 custodyBefore;
        uint256 custodyAfterExec;
        uint256 keeperBefore;
        uint256 keeperAfter;
        uint256 markAfter;
        uint256 withdrawn;
        bool withdrawOk;
        uint8 previewInvalidReason;
        bool previewValid;
    }

    // ------------------------------------------------------------------ per-parameter-set hooks

    /// Other book state before TRADER opens (none at the fixture parameters).
    function _preOpen() internal virtual {}

    function _openMargin() internal pure virtual returns (uint256) {
        return OPEN_MARGIN;
    }

    /// A fill inside the divergence band, in 0.1 bp of adverse move.
    function _bandTenths() internal pure virtual returns (uint256) {
        return 1350; // +1.35%
    }

    /// A fill just below the band, where the withdrawal does not change the outcome.
    function _belowBandTenths() internal pure virtual returns (uint256) {
        return 1250; // +1.25%
    }

    /// The 0.5 bp sweep range, in 0.1 bp of adverse move.
    function _sweepTenths() internal pure virtual returns (uint256 from, uint256 to) {
        return (1200, 1500);
    }

    // ------------------------------------------------------------------ helpers

    function _size(
        address a
    ) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _execute(
        uint64 orderId,
        uint256 fillPrice
    ) internal returns (OrderV3Types.ExecutionResult memory res) {
        // Unique historical tick at commitTime + 1 for the queue head (BasePerpTest._mockHistoricalPublishTime).
        bytes[] memory hist = _mockPythUpdateData(fillPrice);
        vm.prank(KEEPER);
        res = router.executeOrder(orderId, hist);
    }

    function _openViaRouter() internal {
        _preOpen();
        _fundTrader(TRADER, FUNDING);
        vm.prank(TRADER);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, SIZE, _openMargin(), 0, false);
        OrderV3Types.ExecutionResult memory res = _execute(openId, P0);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "setup: open executed");
        assertEq(_size(TRADER), SIZE, "setup: position open");
    }

    /// Ordinary half-close with loose bounds and NO equity floor (not the addMargin/equity-bound lever).
    function _commitPartialClose() internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("card5394-close", router.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = HALF;
        request.marginDelta = 0;
        request.targetPrice = type(uint256).max; // a LONG close accepts any price at or below target
        request.isClose = true;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: 0,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
        vm.warp(vm.getBlockTimestamp() + 10);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(TRADER);
        orderId = router.commitOrder(request);
    }

    function _arm(
        uint256 fillPrice,
        bool withdrawFree
    ) internal returns (Arm memory a) {
        vm.deal(KEEPER, 10 ether);
        _openViaRouter();
        uint64 id = _commitPartialClose();
        IMarginClearinghouse.AccountAssetBuckets memory b = clearinghouse.getAccountAssetBuckets(TRADER);
        a.freeBeforeDecision = b.freeSettlementAssets;
        a.custodyBefore = clearinghouse.balanceAssets(TRADER);
        a.keeperBefore = clearinghouse.balanceAssets(KEEPER);
        if (withdrawFree && b.freeSettlementAssets > 0) {
            // The trader's post-commit action: an ordinary owner withdrawal. No oracle input, no queue check.
            vm.prank(TRADER);
            try clearinghouse.withdraw(TRADER, b.freeSettlementAssets) {
                a.withdrawOk = true;
                a.withdrawn = b.freeSettlementAssets;
            } catch {
                a.withdrawOk = false;
            }
        }
        // The planner's own classification of the same half-close at the same fill, read just before execution.
        ICfdEngineTypes.ClosePreview memory p = engineLens.previewClose(TRADER, HALF, fillPrice);
        a.previewValid = p.valid;
        a.previewInvalidReason = uint8(p.invalidReason);
        OrderV3Types.ExecutionResult memory res = _execute(id, fillPrice);
        a.status = uint8(res.status);
        a.terminalReason = uint8(res.terminalReason);
        a.sizeAfter = _size(TRADER);
        a.custodyAfterExec = clearinghouse.balanceAssets(TRADER);
        a.keeperAfter = clearinghouse.balanceAssets(KEEPER);
        a.markAfter = engine.lastMarkPrice();
    }

    function _pair(
        uint256 fillPrice
    ) internal returns (Arm memory keep, Arm memory kill) {
        uint256 s = vm.snapshotState();
        keep = _arm(fillPrice, false);
        vm.revertToState(s);
        kill = _arm(fillPrice, true);
        vm.revertToState(s);
    }

    /// A LONG profits when the basket mark FALLS (CfdTypes.sol:14-17), so the adverse direction for this close is UP.
    /// `tenthsBp` is the adverse move in 0.1 bp.
    function _adverse(
        uint256 tenthsBp
    ) internal pure returns (uint256) {
        return P0 + (P0 * tenthsBp) / 100_000;
    }

    // ------------------------------------------------------------------ tests

    /// PROPERTY (FAILS on the pinned code): once a close is committed and its fill tick is fixed, whether it executes
    /// must not depend on a choice the trader makes afterwards. Here the choice is an ordinary withdrawal of free
    /// settlement, on an ordinary partial close with no unusual bound.
    function test_Property_WithdrawAfterCommitCannotKillQueuedPartialClose() public {
        uint256 t = _bandTenths();
        (Arm memory keep, Arm memory kill) = _pair(_adverse(t));

        console2.log("adverse fill (0.1 bp)", t);
        console2.log("KEEP (trader does nothing)");
        console2.log("  status (2=Executed,3=Failed) / terminalReason", keep.status, keep.terminalReason);
        console2.log("  trader size after", keep.sizeAfter);
        console2.log("KILL (trader withdraws its free settlement first)");
        console2.log("  status (2=Executed,3=Failed) / terminalReason", kill.status, kill.terminalReason);
        console2.log("  planner previewClose invalidReason", kill.previewInvalidReason);
        console2.log("  trader size after", kill.sizeAfter);
        console2.log("  custody before / withdrawn / custody after", kill.custodyBefore, kill.withdrawn, kill.custodyAfterExec);
        console2.log("  keeper custody before/after", kill.keeperBefore, kill.keeperAfter);

        // Same world and same committed order up to the decision; the arms differ only by the withdrawal.
        assertEq(keep.custodyBefore, kill.custodyBefore, "same custody at decision time");
        assertEq(keep.freeBeforeDecision, kill.freeBeforeDecision, "same free settlement at decision time");
        assertTrue(kill.withdrawOk, "the withdrawal is allowed with the close queued");
        assertEq(keep.markAfter, kill.markAfter, "same fill/mark in both arms");
        // KEEP: the ordinary queued close executes at the fill.
        assertEq(keep.status, uint8(OrderV3Types.LifecycleStatus.Executed), "KEEP executes");
        assertEq(keep.sizeAfter, SIZE - HALF, "KEEP halves the position");
        // KILL: terminal planner rejection, position size unchanged, and the trader's custody plus what it withdrew
        // equals its custody before: the failed attempt debits nothing and pays no executor.
        assertEq(kill.status, uint8(OrderV3Types.LifecycleStatus.Failed), "KILL fails terminally");
        assertEq(kill.terminalReason, uint8(OrderV3Types.TerminalReason.PlannerRejected), "planner rejected");
        assertEq(
            kill.previewInvalidReason,
            uint8(CfdTypes.CloseInvalidReason.PartialActionChargeUncollectible),
            "planner reason is PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE"
        );
        assertEq(kill.sizeAfter, SIZE, "KILL leaves the position size unchanged");
        assertEq(kill.custodyAfterExec + kill.withdrawn, kill.custodyBefore, "KILL debits the trader nothing");
        assertEq(kill.keeperAfter, kill.keeperBefore, "KILL pays no executor");

        assertEq(kill.status, keep.status, "free option: an ordinary withdrawal after commit decides the queued close");
    }

    /// CONTROL: just below the band the same withdrawal does not change the outcome (released margin alone covers the
    /// action charge), so the lever exists only where health limits the release below the charge.
    function test_Control_BelowBandWithdrawalIsInert() public {
        (Arm memory keep, Arm memory kill) = _pair(_adverse(_belowBandTenths()));
        assertTrue(kill.withdrawOk, "control: withdrawal allowed");
        assertEq(keep.status, uint8(OrderV3Types.LifecycleStatus.Executed), "control: keep executes");
        assertEq(kill.status, uint8(OrderV3Types.LifecycleStatus.Executed), "control: kill arm also executes");
    }

    /// CONTROL (harness): at an unchanged fill the close executes in both arms.
    function test_Control_FlatFillExecutesEitherWay() public {
        (Arm memory keep, Arm memory kill) = _pair(P0);
        assertEq(keep.status, uint8(OrderV3Types.LifecycleStatus.Executed), "control: keep executes");
        assertEq(kill.status, uint8(OrderV3Types.LifecycleStatus.Executed), "control: kill arm also executes");
        assertEq(kill.sizeAfter, SIZE - HALF, "control: kill arm halves the position");
    }

    /// Characterization: a 0.5 bp sweep of adverse fills. Logs every fill where the two arms diverge, with the
    /// planner's own close classification, and the band's first and last fill.
    function test_Characterize_SweepBand() public {
        (uint256 from, uint256 to) = _sweepTenths();
        uint256 diverged;
        uint256 firstTenths;
        uint256 lastTenths;
        for (uint256 t = from; t <= to; t += 5) {
            (Arm memory keep, Arm memory kill) = _pair(_adverse(t));
            if (keep.status != kill.status) {
                if (diverged == 0) {
                    firstTenths = t;
                }
                lastTenths = t;
                diverged++;
                console2.log("diverges at adverse fill (0.1 bp)", t);
                console2.log("  keep status/reason/previewReason", keep.status, keep.terminalReason, keep.previewInvalidReason);
                console2.log("  kill status/reason/previewReason", kill.status, kill.terminalReason, kill.previewInvalidReason);
            }
        }
        console2.log("sweep from/to (0.1 bp)", from, to);
        console2.log("diverging fills (0.5 bp grid)", diverged);
        console2.log("band first/last (0.1 bp)", firstTenths, lastTenths);
        assertGt(diverged, 0, "the band exists");
    }
}

/// The same tests at the release risk parameters (packages/perps/DEPLOYMENT.md "Initial release parameters": vpiFactor
/// 0.01e18, maint 10 / init 20 bps; every other RiskParams field equals the fixture). BOB first opens a larger SHORT, so
/// TRADER's LONG is the minority side and its partial close carries a VPI charge rather than a rebate.
contract Card5394S1WithdrawLeverReleaseTest is Card5394S1WithdrawLeverTest {
    address internal constant BOB = address(0xB0B);

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

    /// Release IM is 20 bps (20 USDG on 10k notional); 50 USDG of margin keeps the residual near the health limit.
    function _openMargin() internal pure override returns (uint256) {
        return 50e6;
    }

    function _preOpen() internal override {
        _fundTrader(BOB, 50_000e6);
        vm.prank(BOB);
        uint64 id = router.commitOrder(CfdTypes.Side.SHORT, 30_000e18, 5000e6, 0, false);
        OrderV3Types.ExecutionResult memory res = _execute(id, P0);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "setup: BOB short executed");
    }

    function _bandTenths() internal pure override returns (uint256) {
        return 320; // +0.32%
    }

    function _belowBandTenths() internal pure override returns (uint256) {
        return 250; // +0.25%
    }

    /// 31 fills; one pair is ~24M gas, so this stays under forge's per-test gas budget.
    function _sweepTenths() internal pure override returns (uint256 from, uint256 to) {
        return (250, 400);
    }
}
```

### Repeated-trade measurement

`Card6482S1RepeatedTrade.t.sol` runs lever 1 many times on the same `BasePerpTest` fixture. It uses the release risk parameters from `packages/perps/DEPLOYMENT.md` ("Initial release parameters": `vpiFactor` 0.01e18, maintenance 10 bps, initial 20 bps, `executionFeeBps` 4), set through the fixture's `_riskParams` hook, and the shipped 60 s execution window. Each round runs as follows:

1. Open a LONG through the Router.
2. Commit a partial close of half.
3. The trader decides at commit + 52 s, knowing the fill tick F and the current price D. Both come from the modelled price path; the mocked tick itself is built at execution.
4. Close the rest with a plain close at the next modelled tick, so every round ends flat.

Three arms run on the same seeded price path:

- **INFORMED:** the close carries the unreachable equity bound. The trader calls `addMargin` (keep) iff F < D, since a LONG gains when the mark falls; otherwise it lets the close fail.
- **BLIND:** the same bound and `addMargin`, but the trader keeps or drops on a coin flip that never looks at the price.
- **CONTROL:** the same close with no bound, so it always fills. This is the legitimate baseline: the same trades without the choice.

BLIND is needed because dropping a partial close changes how the exit is split, and that alone changes the fee and VPI split. The execution fee credited on a close is capped by the action charge net of the VPI rebate (`CfdEnginePlanLib.sol:1040-1073`). The option value is INFORMED − CONTROL with BLIND's per-drop effect removed.

Every number is a custody delta: trader custody plus claim, `pool.totalAssets()`, keeper custody and treasury custody. The four sum to zero in every arm. Gas and oracle fees are not included; they would only lower the trader's net.

The price path is a seeded random walk with an **assumed** volatility (365.25-day year, 24 h days). It is not market data. The repository's oracle fixtures hold a few signed Pyth payloads, not a price series, and they need an archival RPC.

| Run | Notional / round | Rounds | Volatility (assumed) | Kept / dropped | Option value to trader | Of which from LPs | Per committed close | INFORMED trader net, all costs |
|---|---|---|---|---|---|---|---|---|
| FX | 10,000 | 50 | 7.5%/yr | 19 / 31 | 11.85 USDG | 11.85 USDG | 0.47 bp | −392.53 USDG (−7.85 / round) |
| FX, second seed | 10,000 | 100 | 7.5%/yr | 45 / 55 | 15.94 USDG | 15.93 USDG | 0.31 bp | −788.27 USDG (−7.88 / round) |
| FX | 100,000 | 50 | 7.5%/yr | 19 / 31 | 118.43 USDG | 118.43 USDG | 0.47 bp | −4,048.59 USDG (−80.97 / round) |
| Stress (not FX) | 10,000 | 50 | 75%/yr | 19 / 31 | 118.12 USDG | 118.07 USDG | 4.72 bp | −214.69 USDG (−4.29 / round) |

The frictionless gross value `σ·√t/√(2π) ≈ 0.399·σ·√t` over the 59 s from fill tick to decision is 0.40 bp at 7.5%/yr and 4.09 bp at 75%/yr. What the runs show:

- The choice moves value from LPs to the trader on committed closes, at about the size the formula predicts, linearly in notional and in volatility. The treasury's share is about zero.
- It is **not** a net-profitable strategy on its own at the shipped settings. The default execution fee alone is 4 bp a side (`CfdEngine.sol:147`), about ten times the option. Fees and the open bounty cost about 7.75 USDG per 10,000-notional round, about 30 times the option value at FX volatility. The INFORMED trader loses money in every run, including the 75%/yr stress run. What LPs lose is the option value on closes a trader makes anyway. This does not prove that no strategy profits: a trader who is closing anyway pays those fees regardless and can exercise selectively.
- Capital: each kept close parks the `addMargin` top-up (50% of position notional in the test; about 20% is the minimum for this bound) until the residual closes. It stays the trader's.
- This measures lever 1 only. Lever 2's value was not measured. Its band is narrow, and the option applies only to fills inside it.

PoC output (10,000 notional, 50 rounds; amounts in 1e6 USDG units; "0.01 bp" lines in hundredths of a basis point):

```console
[PASS] test_Measure_FxVol_10k_N50() (gas: 4540)
Logs:
  ====  FX 7.5%/yr, 10k notional, N=50
  execution window (s) / decision at commit + (s) 60 52
  notional per round (USDG 1e6) / partial close 10000000000 5000000000
  rounds 50
  sigma per sqrt(s), 1e18 = 100% (ASSUMED) 13351000000000
  addMargin top-up per kept close (USDG 1e6), parked until the residual closes 5000000000
  INFORMED (bound; addMargin iff F < D)
    trader net (USDG 1e6)           -392534352
    LPs net, pool.totalAssets (1e6) 5262405
    keeper bounties received (1e6)  10000000
    treasury exec fees (1e6)        377271947
    sum of the four (must be 0)     0
    partial closes kept / dropped   19 31
    notional closed by partials     95000000000
  BLIND (bound; addMargin on a coin flip)
    trader net (USDG 1e6)           -404532637
    LPs net, pool.totalAssets (1e6) 17131661
    keeper bounties received (1e6)  10000000
    treasury exec fees (1e6)        377400976
    sum of the four (must be 0)     0
    partial closes kept / dropped   20 30
    notional closed by partials     100000000000
  CONTROL (same closes, no bound)
    trader net (USDG 1e6)           -408983638
    LPs net, pool.totalAssets (1e6) 17844354
    keeper bounties received (1e6)  10000000
    treasury exec fees (1e6)        381139284
    sum of the four (must be 0)     0
    partial closes kept / dropped   0 0
    notional closed by partials     250000000000
  RAW INFORMED - CONTROL, trader (USDG 1e6)      16449286
  RAW LPs shortfall vs CONTROL (USDG 1e6)        12581949
  RAW treasury shortfall vs CONTROL (USDG 1e6)   3867337
  RAW INFORMED - BLIND, trader (USDG 1e6)        11998285
  OPTION VALUE to trader, adjusted (USDG 1e6)    11849919
    of which from LPs (USDG 1e6)                 11845500
    of which from treasury fees (USDG 1e6)       4419
  OPTION per committed close, 0.01 bp of close   47
  LP loss per committed close, 0.01 bp of close  47
  analytic 0.399 sigma sqrt(W-1), 0.01 bp        40
  INFORMED trader costs per round: treasury + keeper (USDG 1e6) 7745438
  INFORMED trader net per round (USDG 1e6)       -7850687
```

_The repeated-trade test file (`Card6482S1RepeatedTrade.t.sol`) is in the linked secret Gist, because this field is limited to 65,536 characters. Its results are summarised above under "Repeated-trade measurement"; the lever tests above run on their own._

