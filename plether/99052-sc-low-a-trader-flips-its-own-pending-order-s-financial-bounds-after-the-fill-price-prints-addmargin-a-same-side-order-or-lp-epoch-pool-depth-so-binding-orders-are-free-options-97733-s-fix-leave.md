# \#99052 [SC-Low] A trader flips its own pending order's financial bounds after the fill price prints (addMargin, a same-side order or LP-epoch pool depth), so binding orders are free options; #97733's fix leaves every lever open

**Submitted on Nov 11th 2026 at 17:06:53 UTC by @Mclovin for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99052
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

Plether promises that queued orders are binding: "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options" (SECURITY.md:658-659; :366).

The price is pinned at commit: the order fills at the first Pyth tick after the commit second (PletherOracle.sol:610). But the order's own financial bounds (`minPostPositionEquityAssets`, `maxPostPositionSize`, `maxActionChargeAssets`) are checked only at execution, by the policy evaluator (CfdOrderPolicyEvaluatorBase.sol:509-519, :531-537). They are checked against state the trader can still change after that price is public. A miss is a terminal `ConstraintViolation` (OrderRouterExecutionSidecar.sol:879-892) that costs at most the execution bounty.

So a trader commits an order that fails if anyone executes it now, reads the next ticks, and makes it valid only when the pinned price turned out favourable. A keeper can only cancel it.

On the 12 NFP/CPI releases of May-Oct 2026 (real Pyth ticks), with the production 60 s window, an unrelated keeper and a blind long+short commit, the trader nets **+11,512 USDG** and LPs lose **18,259**. The same orders without the switch lose 135,214. If a keeper cancels every order first, the trader loses 29.20 in total.

## Separate from #97733, and #97733's fix leaves it open

My #98219 was grouped under #97733 on its withdraw lever. That lever is #97733's mechanism: VPI-reserve funding at execution draws on free settlement the commit never reserved (`CfdEnginePlanLib._planOpenReserveFunding`, :564-577). The planner rejects it (`PlannerRejected`). This report covers the part that has nothing to do with reserve funding; triage on #98219 already called `addMargin` "the distinct new element".

| | #97733 | This report |
|---|---|---|
| Check that fails | Planner reserve funding (`PlannerRejected`) | Policy evaluator, the order's own bounds (`ConstraintViolation`) |
| Input the trader changes | Free settlement (withdraw) | Position margin, queued same-side size, pool depth |
| Fix | Fund new VPI reserve only from the order's own credit | Freeze bound inputs at commit, or charge for trader-caused failures |

**Measured with #97733's fix applied** (the four-line patch below, `fix97733.diff`, then the same PoC re-run):
- The withdraw lever is closed: `test_Way2_WithdrawAfterCommit` now fills.
- `addMargin` (`test_Way3`), the same-side order (`test_Way1`) and the LP-epoch lever are unchanged to the cent: kept +3,444.71 / +3,443.08 / +3,445.50, dropped −3.76 / −3.71 / −0.20.
- The 12-release straddle is identical: trader +11,512.29, LPs −18,259.16.

**Also not #97906's root.** The option survives a minimum fill delay ("first tick ≥ commit + 3 s"): committing at T−2 pins the same pre-release tick, and `test_Backtest_UnderThreeSecondFillDelay` reproduces the totals exactly.

## Root cause
- **Price fixed at commit:** `minPublishTime = request.commitTime + 1` (PletherOracle.sol:610).
- **Bounds checked at execution, on switchable inputs:**
  1. **`addMargin`** (CfdEngine.sol:501) has no pending-order check. Topping up the pledge lifts equity over the order's own `minPostPositionEquityAssets`.
  2. **A same-side order:** same-side queued opens are allowed (OrderLiquidationBatchLogic.sol:424-428). Filling a small one first pushes the main order past its own `maxPostPositionSize`.
  3. **Pool depth:** anyone may call `OrderRouter.settleLpEpoch(bytes[])`. Activating a pre-requested deposit raises depth, which lowers the open's VPI below its `maxActionChargeAssets`. This lever touches no account state at all, so blocking account actions while an order is pending doesn't close it. It needs an hour-aligned release, so I cite it for the mechanism, not for profit.
- **The spec guards only one bound.** ACCOUNTING_SPEC.md:930-931 prevents owner withdrawals "from manufacturing a settlement-minimum failure" and "does not freeze market-dependent financial bounds". That accepts orders failing because the market moved. Here the order fails or succeeds because the trader acts after seeing the price, which is the free option SECURITY.md:658-659 rules out.

## Attack (addMargin)
1. Hold a 1,100 USDG position in each of two accounts, one long and one short (one side also works).
2. At release second T, commit in each a 2.4M USDG increase with a 60 s window and `minPostPositionEquityAssets` above the current pledge. Both fills are pinned to the T+1 tick. No view of direction is needed.
3. If a keeper executes first, the order fails terminally, costing a few USDG.
4. At T+2.5, read Pyth's T+2 tick. If it moved ≥ 3 bps in one side's favour, call `addMargin` on that account. The next keeper fills it at the T+1 price, and the other order dies when touched.
5. Exit with a close committed at T+10, filled at T+11.

## Impact (12 releases, USDG, trader net after fees, VPI and carry)
| Variant | Trader | LPs | Filled |
|---|---|---|---|
| One side LONG, keeper at T+3 or T+6 | +7,336 | −10,230 | 3/12 |
| One side SHORT | +4,186 | −8,039 | 4/12 |
| Blind long+short | **+11,512** | **−18,259** | 7/12 |
| Blind long+short, all six feeds USD-wide | +21,865 | −28,612 | 7/12 |
| Keeper executes everything at T+1 (worst case) | −29.20 | +0.95 | 0/12 |
| Control: same orders without the switch | −135,214 | +89,093 | 12/12 |

- **Risk profile:** it's an edge, not a sure win. One filled release lost 2,829 when the move reversed after T+2. To make the strategy unprofitable, keepers would have to cancel first on 99.8% of orders.
- **Who can do it:** any trader. No keeper, owner or guardian role is involved.

## Known issues
- **Contradicts:** SECURITY.md:658-659 and :366 ("Users cannot cancel queued orders once committed").
- **The Router and keeper limitations** (SECURITY.md:1343-1345) accept that users *cannot* cancel. They say nothing about a trader switching an order's validity after commit.
- **Pre-audit SC-M4** ("Risk Free Option in 60Sec Oracle Window", fixed in #24): the first-unique-tick fix pins the price, but validity can still be switched afterwards, so that fix is incomplete.
- **Not in** the Cecuro audit, the AI scan, the collaborative audit or the Cantina report.

## Recommended fix
Evaluate `minPostPositionEquityAssets`, `maxPostPositionSize` and `maxActionChargeAssets` against inputs fixed at commit: the margin and queue state then, and the depth then. Alternatively, charge any terminal failure of a pending open a fee that removes the option (about 6 bps of notional on this data). An account-scoped freeze (blocking `addMargin` or same-side commits while an order is pending) leaves the pool-depth lever.



## Link to Proof of Concept

https://gist.github.com/chj2934/ced56ef2eff9a1b9bff2a6d1d1cbd318

## Proof of Concept

## Proof of concept

PoC (secret gist, two test files plus `fix97733.diff`): https://gist.github.com/chj2934/ced56ef2eff9a1b9bff2a6d1d1cbd318

Save both `.t.sol` files in `packages/perps/test/poc/order-bounds/` and run from `packages/perps`:

```
FOUNDRY_TEST=test/poc/order-bounds forge test --dynamic-test-linking --match-path "test/poc/order-bounds/*.t.sol" -vv --gas-limit 100000000000
```

Unmodified code: 17 passed (`OrderBoundsSwitchTest` 10/10, `LpEpochSwitchTest` 7/7).

To test against #97733's fix, run `git apply fix97733.diff` from the repository root and re-run. Exactly one test fails: `test_Way2_WithdrawAfterCommit` (the withdraw lever now fills, which shows the patch works). Every other test passes with identical totals.

| Test | Shows |
|---|---|
| `test_Way3_AddMarginAfterCommit` / `test_Way1_SameSideGuardOrder` | Keeper first: cancel only; kept +3,444.71 / +3,443.08; dropped −3.76 / −3.71; terminal reason `ConstraintViolation` |
| `test_Way2_WithdrawAfterCommit` | #97733's lever, for contrast: terminal reason `PlannerRejected` |
| `test_Backtest_Straddle`, `_OneSide`, `_UsdWide`, `_KeeperFirst_WorstCase`, `_UnderThreeSecondFillDelay` | The 12-release totals above |
| `test_Control_KeeperFillsBlindAtT1`, `test_Control_SameRuleWithoutGate` | The same orders without the switch |
| `LpEpochSwitchTest.test_LpSwitch_ThreeBranches` | Pool-depth lever: cancel only / kept +3,445.50 / dropped −0.20; `ConstraintViolation` (ActionCharge) |

The release oracle is installed by the documented owner rotation in `setUp`, with real Pyth EUR/USD and USD/JPY ticks from Benchmarks and the other four feeds held constant. After `setUp`, every prank is the trader, the trader's LP wallet, or an unrelated keeper.

