# \#98219 [SC-Low] A trader can make its committed order valid or invalid after seeing later prices (addMargin, withdraw, same-side order), so binding orders become free options even with a 60 s window and keeper-only execution

**Submitted on Nov 6th 2026 at 02:24:01 UTC by @Mclovin for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98219
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary
Plether promises that queued orders are binding: "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options" (SECURITY.md:658-659); "Users cannot cancel queued orders once committed" (:366).

The order's price is pinned at commit: it fills at the first Pyth tick after the commit second (PletherOracle.sol:610). But whether the order is allowed to execute is decided later, at execution, from account state the trader can still change. So a trader can:
1. commit an order that fails if anyone executes it now;
2. wait for prices published after the pinned tick;
3. then decide whether to make the order valid.

A keeper can only cancel it, so the trader holds the choice.

There are three ways to switch validity: `addMargin`, withdrawing free settlement, and a same-side order. They need none of the levers my report #97906 uses: no short window, no self-execution, and no information at commit.

On the same 12 NFP/CPI releases as #97906 (May-Oct 2026, real Pyth ticks), with the production 60 s window, an unrelated keeper and a blind long+short commit:
- the trader nets **+11,512 USDG** and LPs lose **18,259**;
- the same orders without the switch lose 135,214;
- if a keeper cancels every order before the trader acts, the trader loses 29.20 in total.

## Root cause
- **Price fixed at commit:** `minPublishTime = request.commitTime + 1` (PletherOracle.sol:610).
- **Validity checked at execution:** `maxPostPositionSize` and `minPostPositionEquityAssets` are evaluated against the
  live account (CfdOrderPolicyEvaluatorBase.sol:517-519, :532-536). A miss is a terminal failure
  (OrderRouterExecutionSidecar.sol:879-892) and costs the trader at most the 0.20 USDG execution bounty.
- **The trader controls those inputs after commit:**
  1. **`addMargin`** (CfdEngine.sol:501-529) has no check for pending orders. Topping up the position's pledge
     lifts equity over the order's own floor.
  2. **Withdraw.** The open's VPI-rebate reserve is funded from free settlement at execution
     (CfdEnginePlanLib.sol:546-576). Withdrawing it makes the open fail (PlannerRejected, code 10); depositing it
     back makes it valid again. The withdraw floor only protects flat accounts with a declared floor
     (CfdEngineSettlementSidecar.sol:261-272).
  3. **Same-side order.** Same-side queued opens are allowed (OrderLiquidationBatchLogic.sol:424-428). Filling a
     small one first pushes the main order past its own `maxPostPositionSize`.
- **The team already guards one case of this.** ACCOUNTING_SPEC.md:930-931: "This protection prevents owner
  withdrawals from manufacturing a settlement-minimum failure". The equity floor, post-position size and
  reserve funding are not covered.

## Attack (addMargin version)
1. Hold a 1,100 USDG position in each of two accounts, one long and one short. One side also works.
2. At release second T, commit in each account a 2.4M USDG increase with a 60 s window, and set
   `minPostPositionEquityAssets` above the current pledge. Both fills are pinned to the T+1 tick. No view of
   direction is needed.
3. If a keeper executes first, the order fails terminally. That costs a few USDG at most.
4. At T+2.5, read Pyth's T+2 tick. If it moved at least 3 bps from the pinned price in one side's favour, call
   `addMargin` on that account. The next keeper fills that order at the T+1 price, and the other order dies
   when touched.
5. Exit: commit a close at T+10, filled at T+11 (the same accounting as #97906's PoC).

## Impact (12 releases, USDG, trader net after fees, VPI and carry)
| Variant | Trader | LPs | Filled |
|---|---|---|---|
| One side LONG, keeper at T+3 or T+6 | +7,336 | −10,230 | 3/12 |
| One side SHORT, keeper at T+3 or T+6 | +4,186 | −8,039 | 4/12 |
| Blind long+short, keeper at T+3 or T+6 | **+11,512** | **−18,259** | 7/12 |
| Blind long+short, all six feeds USD-wide | +21,865 | −28,612 | 7/12 |
| Keeper executes everything at T+1 (worst case) | −29.20 | +0.95 | 0/12 |
| Control: same orders without the switch | −135,214 | +89,093 | 12/12 |

- It is an edge, not a sure win. One filled release lost 2,829 (08-12 CPI, the move reversed after T+2). Net over
  the 12 is clearly positive.
- When a keeper cancels first, each order costs 0.31-3.87 USDG.
- To make the strategy unprofitable, keepers would have to cancel first 99.8% of the time.

## Why this is not #97906
- **What #97906 needs:** the first tick after a release-second commit is stale, a short trader-chosen window and
  self-execution, with the headline in hand at commit.
- **What this needs:** none of those. It even works with a commit at T−2: today's code fills at the T−1 tick, a
  pre-release price with no oracle lag involved, and the blind long+short nets +11,771.
- **Fixes to #97906 that leave this live:**
  - a minimum window;
  - keeper-only execution;
  - any minimum settlement delay ("first tick ≥ commit + 3 s" emulated: identical totals to the cent);
  - a commit blackout around releases;
  - expiry penalties (these are constraint or planner failures, not expiries).
- **What stops it:** only fixing order validity itself, or filling at the execution-second tick. The latter reopens
  keeper tick choice.
- **Relation to my earlier follow-up:** my follow-up 6 on #97906 used the same-side order to make #97906
  keeper-proof. This report covers the general root, adds the two other paths, and shows the fix results above.

## Docs and known issues
- **Contradicts:** SECURITY.md:658-659, :366 ("Binding intents | Users cannot cancel queued orders once committed"),
  :719 and :1345.
- **SECURITY.md:726-727 and :1367-1369** say constraint and planner failures are terminal. That is the building
  block. It does not say the trader may switch an order's validity after commit.
- **ACCOUNTING_SPEC.md:930-931** treats an owner-made failure as something to prevent. Its guard covers only the
  settlement-minimum bound.
- **Pre-audit SC-M4** ("Risk Free Option in 60Sec Oracle Window", fixed in #24): the first-unique-tick fix pins the
  price, but validity can still be switched afterwards, so that fix is incomplete.
- **Not in** the Cecuro audit, the AI scan, the collaborative audit or the Cantina report.

## Suggested fix
- Make the financial bounds and planner funding fixed at commit: evaluate `minPostPositionEquityAssets`,
  `maxPostPositionSize` and the VPI-reserve funding against what the account had committed. Alternatively, block
  `addMargin`, withdrawals and same-side commits from changing a pending order's validity while it is queued.
- At minimum, charge a fee on trader-caused terminal failures large enough to remove the option: at least about
  6 bps of notional on this data.

## Link to Proof of Concept

https://gist.github.com/chj2934/6958415fe3046fd3f4c2c2d2fae7ca4d

## Proof of Concept

## Proof of concept
PoC (secret gist): https://gist.github.com/chj2934/6958415fe3046fd3f4c2c2d2fae7ca4d

Save as `packages/perps/test/poc/own-order-cancel/OwnOrderCancel.t.sol` and run from `packages/perps`:

```
FOUNDRY_FFI=false forge test --match-contract OwnOrderCancelTest -vv --gas-limit 100000000000
```

It uses:
- release risk params and the 12 releases' real Pyth ticks (the same data as #97906's PoC);
- public trader calls only (`commitOrder`, `addMargin`, clearinghouse `withdrawMargin` / `deposit`);
- an unrelated keeper (`executeOrder`);
- no storage writes.

Result: 10 passed.

| Test | Shows |
|---|---|
| `test_Way1_SameSideGuardOrder`, `test_Way2_WithdrawAfterCommit`, `test_Way3_AddMarginAfterCommit` | For each path on 2026-09-04: a keeper at T+1 can only cancel (Failed); a kept LONG is filled by a keeper at T+3 at the pinned T+1 tick (+3,443 / +3,463 / +3,445); a dropped SHORT never fills (−3.71 / −0.20 / −3.76) |
| `test_Backtest_OneSide`, `test_Backtest_Straddle` | The table above |
| `test_Backtest_KeeperFirst_WorstCase` | A keeper at T+1 on every release: −15.69 / −13.49 / −29.20 |
| `test_Backtest_UnderThreeSecondFillDelay` | An emulated 3 s settlement delay: identical totals |
| `test_Backtest_UsdWide` | All six feeds move with EUR: +12,798 / +9,076 / +21,865 |
| `test_Control_KeeperFillsBlindAtT1`, `test_Control_SameRuleWithoutGate` | Without the switch the same orders lose |

