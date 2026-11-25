# \#99813 [SC-Low] A skew-reducing open funds its VPI reserve from withdrawable free settlement, so withdrawing after the fill tick cancels the open and binding opens become free options

**Submitted on Nov 15th 2026 at 11:33:16 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99813
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
A fresh open that reduces skew earns a VPI rebate and must back a larger negative-VPI reserve. The planner funds that reserve from the account's free settlement first and only otherwise from the new position's pledge, then checks initial margin on what is left. If the committed margin sits inside a band up to one execution fee wide, the open fills when the trader keeps free settlement and fails terminally (`PlannerRejected`) when the trader withdraws it before execution. `withdrawMargin` is allowed while the open is pending. The fill price is fixed at the first tick after commitment, so the trader decides whether the queued open fills after that price, and several seconds of later prices, are public. No signed bound and no existing position are needed, and a cancelled open costs at most the 0.20 USDG open bounty.

## Vulnerability Details
1. Reserve sourcing depends on live free settlement. `_planOpenReserveFunding` takes the reserve increase from `resultingFreeSettlementAssets` and charges the rest to the new pledge ([CfdEnginePlanLib.sol L561-L579](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L561-L579)):

```solidity
uint256 vpiReserveFromFreeAssets = vpiReserveIncreaseAssets < openCostPlan.resultingFreeSettlementAssets
    ? vpiReserveIncreaseAssets
    : openCostPlan.resultingFreeSettlementAssets;
delta.vpiRebateReserveFromPledgeAssets = vpiReserveIncreaseAssets - vpiReserveFromFreeAssets; // <-@ depends on live free settlement
...
delta.positionMarginAfterOpen =
    openCostPlan.resultingPositionMarginAssets - delta.vpiRebateReserveFromPledgeAssets - reserveIncreaseAssets;
```

   The reserve target is the gross rebate, while only the rebate net of the fee is credited. With no free settlement, the fee-sized difference comes out of the pledge.

2. Initial margin is then checked on that pledge ([CfdEnginePlanLib.sol L429-L436](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L429-L436)). `INSUFFICIENT_INITIAL_MARGIN` is a user-invalid terminal execution failure ([CfdEnginePlanLib.sol L113-L130](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L113-L130); [OrderRouterExecutionSidecar.sol L564-L576](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L564-L576), [L834-L852](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L834-L852)). Committed margin and the bounty stay reserved; free settlement does not.

3. Withdrawals ignore a pending open. For a flat account, `validateWithdraw` enforces only a positive settlement floor ([CfdEngineSettlementSidecar.sol L255-L272](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L255-L272); [MarginClearinghouse.sol L416-L478](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L416-L478); [OrderLiquidationBatchLogic.sol L302-L311](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L302-L311)). The spec states that guard's purpose: "This protection prevents owner withdrawals from manufacturing a settlement-minimum failure" ([ACCOUNTING_SPEC.md L930](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L930)). Here a withdrawal manufactures an initial-margin failure instead, and that failure does not depend on the market.

4. The fill is the first unique tick after `commitTime` ([PletherOracle.sol L610-L622](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L622)), however late the order executes.

Attack:
- Commit the skew-reducing open with margin inside the band (computable from the commit-time price), and withdraw free settlement in the same transaction.
- Left alone, any execution fails with `PlannerRejected`, and the keeper takes the bounty.
- If the move after the fill tick favours the open, deposit and execute it yourself: it fills at the old tick.

The switch does not need a withdrawal after commitment. A later own open can do the same job: its committed margin locks the remaining free settlement, and it then fails on its own bound at the cost of its bounty. A deposit, or a `depositFor` from another address the trader controls, switches the open back on. In the PoC, the open as committed fails with `PlannerRejected`; after `depositFor(5,000 USDG)` from another address it fills.

On the public Arbitrum Sepolia v2.0.0 deployment, the reference keeper executed all 278 live orders 0-8 s after the fill tick's publish second. It polls every 5 s when idle, so a commit just after a poll leaves 7-8 s to decide.

The pre-audit reported this impact as SC-M4 ("Risk Free Option in 60Sec Oracle Window", fixed): the losing queued open was cancelled after commit by a withdrawal (an LP withdrawal causing `SOLVENCY_EXCEEDED` -> refund). The fix removed that refund path, but the fix at a82d4fef is incomplete. A withdrawal after commit still cancels the losing open, now through the trader's own `withdrawMargin` and the planner's sourcing of the VPI reserve from live free settlement.

Contradicted text: [SECURITY.md L658-L659](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L658-L659) ("binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options").

## Impact Details
**Low.** The stated security property does not hold, and LPs pay the option value.

PoC: Foundry, unmodified contracts, public entry points only, release parameters (fee 4 bp, VPI 1%, IM 0.2%, MM 0.1%, open bounty 1 bp in [0.01, 0.20] USDG, 60 s window, 10M Junior + 40M Senior, six release feeds). Price path: the real Pyth six-feed basket of 14 Oct 2026 12:10-12:49 UTC (US data at 12:30, -23 bp in 18 s), interpolated per second. An honest account holds 2.9M USDG of exposure, so the trader's 961k USDG open reduces skew. The control makes the same trades with ordinary binding orders committed at the decision time.

One open (12:29:50 UTC, 961k LONG, committed margin 3,077.99 USDG):
- free settlement kept: `Executed` (pledge 2,116.12, reserve 462.60);
- free settlement withdrawn: `Failed / PlannerRejected`, cost 0.20 USDG.

| Run | Exercised | Trader vs control | LPs vs control |
|---|---|---|---|
| SHORT-heavy book (the LONG open reduces skew), commits timed after the keeper's poll, 39 min, decide at 7 s, exercise above the 8.5 bp round trip | 1 (12:30 release, edge 8.93 bp) | +955.63 USDG (+300.02 net vs -655.61) | -981.82 |
| Trader opens anyway, keeps every favourable fill, cancels the rest; measured keeper delays, 25 min, decide at 3 s | 68 | +2,095.82 | -2,120.67 |
| Same, commits timed after the keeper poll, 39 min, decide at 6 s | 103 | +4,630.34 | -4,632.19 |

- A trader that opens anyway enters 0.32-0.47 bp better per open, and LPs lose the same amount.
- No view on direction is needed. An unused attempt costs at most one bounty, and nothing when the trader executes its own order.
- Attempts can be repeated every few seconds whenever the book is skewed.

## Recommendation
Fund an open's negative-VPI reserve only from value reserved at commitment. Applying the settlement-floor withdrawal predicate to pending opens is not enough on its own, because another own order or a `depositFor` can move free settlement without a withdrawal.

## References
- [CfdEnginePlanLib.sol L561-L579](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L561-L579), [L429-L436](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L429-L436), [L113-L130](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L113-L130)
- [CfdEngineSettlementSidecar.sol L255-L272](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L255-L272)
- [MarginClearinghouse.sol L416-L478](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L416-L478)
- [OrderRouterExecutionSidecar.sol L564-L576](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L564-L576), [L834-L852](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L834-L852)
- [OrderLiquidationBatchLogic.sol L302-L311](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L302-L311)
- [PletherOracle.sol L610-L622](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L622)
- [SECURITY.md L658-L659](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L658-L659), [ACCOUNTING_SPEC.md L917-L930](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L917-L930)
- Pre-audit SC-M4: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf

## Proof of Concept

## Proof of Concept

The full Foundry tests and their run output are in this gist: https://gist.github.com/tinnohofficial/6429b22cb04ae17c7645ce0140d81c24
- `WithdrawFillSwitch.t.sol`: 12 tests
- `WithdrawFillSwitchNoWithdrawal.t.sol`: 13 tests
- `*.output.txt`: the output of each suite

The tests use the repository's own harness (`BasePerpTest`), its Pyth mock and the six release basket feeds. They call public entry points only (`commitOrder`, `executeOrder`, `updateMarkPrice`, `withdrawMargin`, `depositFor`), and no in-scope contract is modified.

### How to run
1. Check out `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` and install the dependencies as in the repository README.
2. Copy both test files into `packages/perps/test/poc/`.
3. From the repository root, run:

```
forge test --root packages/perps --match-path test/poc/WithdrawFillSwitch.t.sol -vv
forge test --root packages/perps --match-path test/poc/WithdrawFillSwitchNoWithdrawal.t.sol -vv
```

The suites pass 12/12 and 13/13, and repeated runs give identical output.

### Step by step: one open, two outcomes (`test_Switch_OneOpenTwoOutcomes`)
1. **Setup:** release parameters, a 10M Junior and 40M Senior pool, and an honest account holding a 2.9M USDG SHORT, so a LONG open reduces skew and earns a VPI rebate.
2. **Commit:** at 12:29:50 UTC on the real 14 Oct 2026 path, a flat trader commits a 961k USDG LONG with 3,077.99 USDG of committed margin. That is inside the band where the reserve sourcing decides the initial-margin check.
3. **Prices:** the fill tick is 0.961877. Seven seconds later the price is 0.961526, so the LONG is 3.64 bp in profit at the fill price.
4. **Control (free settlement kept):** the open ends `Executed`, with a position margin of 2,116.12 USDG and a negative-VPI reserve of 462.60 USDG. The log line is "free settlement kept: status / reason 2 1".
5. **Attack branch (free settlement withdrawn at decision time):** the trader calls `withdrawMargin` before execution, and the same open ends `Failed / PlannerRejected`, because the planner now takes the reserve from the pledge and initial margin fails. The log lines are "free settlement withdrawn: status / reason 3 7" and "trader cost of the cancelled open (USDG atoms) 200000", i.e. 0.20 USDG.

### Step by step: no withdrawal (`test_Switch_NoWithdrawal_LaterOrderLocksFreeSettlement`)
1. The trader commits the open, then commits a second own order whose committed margin locks the remaining free settlement. Its execution-notional bound guarantees it fails.
2. As committed, the open fails `PlannerRejected` ("as committed: open status / reason 3 7").
3. After `depositFor(5,000 USDG)` from another address the trader controls, the open fills ("after depositFor: open status / reason 2 1"), and the locking order fails on its bound ("locking order status / reason 3 8").

### Real-path results (14 Oct 2026 Pyth basket; trader vs control, LPs vs control, USDG)
- SHORT-heavy book, commits timed after the keeper poll, exercised only above the round-trip cost, decide at 7 s: +955.63 / -981.82. The trader ends +300.02 USDG net, while the control loses 655.61.
- Opens anyway, measured keeper delays, decide at 3 s: +2,095.82 / -2,120.67
- Opens anyway, commits timed after the keeper poll, decide at 6 s: +4,630.34 / -4,632.19

Each `*_Control` test makes the same trades with ordinary binding orders committed at the decision time.
