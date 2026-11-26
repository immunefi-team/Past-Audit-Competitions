# \#99811 [SC-Low] Signed post-trade bounds plus `addMargin` turn binding queued orders into free options against LPs

**Submitted on Nov 15th 2026 at 11:32:17 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99811
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
A trader can commit an increase or a partial close whose signed `maxPostLeverageBps` or `minPostPositionEquityAssets` the committed margin cannot meet. If nobody acts, the order fails terminally with `ConstraintViolation`. That costs nothing for a close, and at most the 0.20 USDG open bounty for an increase. If the trader calls `CfdEngine.addMargin` before execution, the same order fills at its committed tick. So the trader decides whether the order fills after the fill price, and several seconds of later prices, are public. No signed bound is needed: with the default unlimited bounds, two overlapping queued increases give the same switch through the initial-margin check. With two accounts a trader can commit a LONG and a SHORT increase in the same second and switch on only the leg the later move favours. A trader that closes anyway can exit only when the price moves against it after the fill tick. SECURITY.md states the opposite as a security property: "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options".

## Vulnerability Details
1. The bounds are checked at execution against live position state. Post-trade equity is projected from the position's current margin ([`CfdOrderPolicyEvaluatorBase.sol` L367-L386](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L367-L386) for opens, [L455-L472](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L455-L472) for closes) and enforced at [L526-L539](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L526-L539).
2. `CfdEngine.addMargin` has no pending-order restriction; it only requires an open position ([`CfdEngine.sol` L501-L529](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L529)).
3. Nothing at commitment checks that these bounds are reachable. The protocol hardened only the other trader-set floor: a positive `minPostSettlementBalanceAssets` must be funded at commitment, withdrawals must preserve it, and no other order may overlap it ([`OrderLiquidationBatchLogic.sol` L302-L311](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L302-L311); [ACCOUNTING_SPEC.md L917-L928](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L917-L928)).
4. A missed bound is a terminal, allow-listed failure ([`OrderRouterExecutionSidecar.sol` L564-L576](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L564-L576), [L879-L893](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L879-L893)). Its only cost is the open bounty: at most 0.20 USDG, and nothing when the trader executes its own order. Closes carry no bounty.
5. The fill is fixed at the first unique tick after `commitTime` ([`PletherOracle.sol` L610-L622](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L622)), however late the order executes.

**The switch does not need a signed bound.** With unlimited bounds (the web app's defaults), a trader commits a minimum-size increase and then the real increase in one transaction. The real increase's margin relies on the position's free equity. The commit-time check counts that equity for both orders, because it previews each open against the live position and projects earlier queued orders only for their side ([`OrderLiquidationBatchLogic.sol` L423-L446](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L423-L446)). At execution the small increase fills first, and the large one fails terminally with `INSUFFICIENT_INITIAL_MARGIN` ([`CfdEnginePlanLib.sol` L426-L437](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L426-L437)). After `addMargin`, it fills at the committed tick instead. An unused attempt costs about 1 USDG.

The decision window is the honest keeper's execution delay. On the public Arbitrum Sepolia v2.0.0 deployment, the reference order keeper executed all 278 live-market orders 0-8 s after the fill tick's publish second (median 4 s), measured as `terminalTime - oraclePublishTime` from the authenticated `OrderFinalized` receipts. The keeper polls every 5 s when its queue is empty (every 1 s once it has pending work), so a trader that commits into an idle queue just after a poll gets about 7-8 s.

The pre-audit reported the same impact as SC-M4 ("Risk Free Option in 60Sec Oracle Window", fixed): a straddle whose losing leg is cancelled for free. Its root was keeper tick choice inside a 60 s window plus a refund path triggered by an LP withdrawal, and the fix removed both. The root here is different: post-trade bounds and initial margin are re-checked against live margin that the trader can still change. So the fix at a82d4fef is incomplete: the free option it was meant to remove is still reachable through this mechanism.

Contradicted text: [SECURITY.md L658-L659](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L658-L659) (quoted above).

## Impact Details
**Low.** The stated security property does not hold, and LPs pay the option value.

PoC: Foundry, unmodified contracts, public entry points only, release parameters (fee 4 bp per fill, VPI 1%, IM 0.2%, MM 0.1%, open bounty 1 bp in [0.01, 0.20] USDG, 60 s window, 10M Junior + 40M Senior). Price path: the real Pyth six-feed basket of 14 Oct 2026 12:10-12:49 UTC (US data at 12:30, -23 bp in 18 s), linearly interpolated per second between the real samples. Each attacker trade is about 961k USDG notional. The control makes the same trades at the same times with ordinary binding orders committed at the attacker's decision time.

| Run (39 minutes) | Exercised | Attacker vs control | LPs vs control |
|---|---|---|---|
| Close option on held positions, measured keeper delay, decide 3 s after the fill tick | 208 closes (161 attempts lost to the keeper) | +3,275.93 USDG | -3,276.22 USDG |
| Same, commits timed after the keeper's poll, decide at 6 s | 209 closes | +6,378.27 USDG | -6,379.20 USDG |
| Two-account straddle, switch on the winner only when its edge exceeds the 8.1 bp round-trip cost, decide at 7 s | 1 leg, at the 12:30 release | +928.84 USDG | -980.83 USDG |
| **No signed bound** (overlapping increases plus `addMargin`): increase anyway, keep favourable fills, measured keeper, decide at 3 s, 25 min | 65 increases | +1,968.02 USDG | -2,236.31 USDG |
| Same, commits timed after the keeper's poll, decide at 6 s, 25 min | 69 increases | +3,147.63 USDG | -3,271.67 USDG |
| Same, exercise only above the 8.5 bp round trip, decide at 7 s, 39 min | 1, at the 12:30 release | +723.53 USDG | -982.58 USDG |

- The close option costs nothing extra. A trader that closes anyway exits 0.16-0.32 bp better per close, and LPs lose the same amount.
- The straddle needs no view on direction, and an unused attempt costs at most 0.40 USDG. The one exercise at the release left the attacker **+270.97 USDG net, after all fees and after 51.60 USDG of bounties paid to the keeper** for the 129 attempts it executed first. The control trader lost 657.87 USDG on the same trade.
- Attempts are independent and can be repeated every few seconds; the PoC runs 162-378 attempts per path.

## Recommendation
Reject `addMargin` while the account has any pending open or increase. Alternatively, evaluate initial margin, `minPostPositionEquityAssets` and `maxPostLeverageBps` against the margin recorded at commitment, so margin added later cannot satisfy them. In either case, also project earlier queued opens in the commit-time admission check.

## References
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L526-L539
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L367-L386
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L455-L472
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L529
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L302-L311
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L423-L446
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L426-L437
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L564-L576
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L879-L893
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L622
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L658-L659
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L917-L928
- Pre-audit SC-M4 "Risk Free Option in 60Sec Oracle Window": https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf

## Proof of Concept

## Proof of Concept

The full Foundry tests and their run output are in this gist: https://gist.github.com/tinnohofficial/d0d63e49eeb14471a2f66cab9c94e4a5
- `QueuedOrderFillSwitch.t.sol`: 12 tests
- `UnboundedIncreaseFillSwitch.t.sol`: 10 tests
- `*.output.txt`: the output of each suite

The tests use the repository's own harness (`BasePerpTest`), its Pyth mock and the six release basket feeds. They call public entry points only (`commitOrder`, `executeOrder`, `updateMarkPrice`, `CfdEngine.addMargin`), and no in-scope contract is modified.

### How to run
1. Check out `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` and install the dependencies as in the repository README.
2. Copy both test files into `packages/perps/test/poc/`.
3. From the repository root, run:

```
forge test --root packages/perps --match-path test/poc/QueuedOrderFillSwitch.t.sol --threads 4 --gas-limit 9000000000000000000 -vv
forge test --root packages/perps --match-path test/poc/UnboundedIncreaseFillSwitch.t.sol --threads 4 --gas-limit 9000000000000000000 -vv
```

The raised gas limit is needed because the real-path tests replay 25-39 minutes of second-by-second execution (up to 6.75B gas). The two suites pass 12/12 and 10/10, and repeated runs give identical output.

### Step by step: the switch (`test_Switch_StraddleKeepsOnlyTheWinningLeg`)
1. **Setup:** release parameters, a 10M Junior and 40M Senior pool, and two accounts of the same trader, one holding a LONG and one a SHORT.
2. **Commit:** at 12:29:50 UTC on the real 14 Oct 2026 path, the LONG account commits an increase and the SHORT account commits an increase, each with a `maxPostLeverageBps` its committed margin cannot meet.
3. **Prices:** the fill tick (the first update after the commit) is 0.961877. Six seconds later the price is 0.961649, so at the fill tick the LONG leg is already 2.37 bp in profit.
4. **Control:** executed as committed, both orders end `Failed / ConstraintViolation` (status 3, reason 8).
5. **Attack:** the trader calls `addMargin` on the LONG account only, then executes both orders. The LONG ends `Executed` (status 2) at the committed tick, and the SHORT ends `Failed / ConstraintViolation`.

### Step by step: no signed bound (`UnboundedIncreaseFillSwitch.t.sol`)
1. With unlimited bounds, the trader commits a minimum-size increase and then a 961k USDG increase in one transaction. The large increase's margin relies on the position's free equity, and both orders pass the commit-time check.
2. Executed as committed, the small increase fills and the large one fails `INSUFFICIENT_INITIAL_MARGIN`.
3. If the price after the fill tick favours the increase, the trader calls `addMargin` before execution, and the increase fills at the committed tick.

### Real-path results (14 Oct 2026 Pyth basket; attacker vs control, LPs vs control, USDG)
- Close option, measured keeper delay, decide at 3 s: +3,275.93 / -3,276.22
- Close option, commits timed after the keeper poll, decide at 6 s: +6,378.27 / -6,379.20
- Two-account straddle, exercised only above the round-trip cost, decide at 7 s: +928.84 / -980.83. The attacker ends +270.97 USDG net after all fees and bounties, while the control trader loses 657.87.
- No signed bound, measured keeper, decide at 3 s: +1,968.02 / -2,236.31
- No signed bound, timed keeper, decide at 6 s: +3,147.63 / -3,271.67

Each `*_Control` test makes the same trades with ordinary binding orders committed at the attacker's decision time.
