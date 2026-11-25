# \#99926 [SC-Low] Settling a matured LP epoch before execution cancels a queued open for free, so binding opens become options against LPs

**Submitted on Nov 15th 2026 at 18:38:26 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99926
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
A queued open's VPI, and with it the initial-margin and skew checks, is recomputed at execution against the live pool depth `HousePool.totalAssets()`. Funding a matured LP redemption lowers that depth. LP epoch settlement is permissionless and is a separate call from order execution. So a trader commits a skew-increasing open in the last second before an hourly LP boundary, with margin inside a band, and decides after the fill tick is public:
- execute the open first, and it fills at its committed tick;
- settle the matured epoch first, and the same open fails terminally (`PlannerRejected`) at no cost.

The redemption can be any LP's, including the trader's own. SECURITY.md states the opposite as a security property: "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options".

## Vulnerability Details
1. Execution samples the live pool depth ([OrderRouterExecutionSidecar.sol L953](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L953)) and prices VPI with it ([OpenAccountingLib.sol L127-L129](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OpenAccountingLib.sol#L127-L129)):

```solidity
request.poolDepthAssets = IHousePool(engine_.pool()).totalAssets(); // <-@ live depth at execution time
```

   Skew cost is inversely proportional to depth ([CfdMath.sol L118-L180](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdMath.sol#L118-L180)). The skew cap is `0.4 x depth` ([CfdEnginePlanLib.sol L446-L480](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L446-L480)), and initial margin is checked after costs ([L426-L437](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L426-L437)). Commitment admits the open at the depth of the commit block ([OrderLiquidationBatchLogic.sol L430-L446](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L430-L446)).

2. Funding a matured redemption moves assets out of the pool ([HousePool.sol L856-L862](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L856-L862), [L1004-L1006](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1004-L1006)). Anyone can trigger it through `OrderRouter.settleLpEpoch` ([OrderRouter.sol L479-L484](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L479-L484); [OrderLiquidationBatchLogic.sol L549-L589](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L549-L589)), in any order relative to `executeOrder`.

3. Both resulting failures are terminal. `INSUFFICIENT_INITIAL_MARGIN` is user-invalid and `SKEW_TOO_HIGH` is a protocol-state invalidation ([CfdEnginePlanLib.sol L112-L128](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L112-L128)). Either one finalizes the order as `PlannerRejected` ([OrderRouterExecutionSidecar.sol L564-L576](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L564-L576), [L834-L852](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L834-L852)). Committed margin returns to the trader, and the bounty goes to the executor, which is the trader.

4. The fill is the first unique tick after `commitTime` ([PletherOracle.sol L610-L622](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L622)), however late the order executes.

Attack, at any hourly boundary `h` with a matured redemption:
- At `h - 1`, commit a skew-increasing open whose margin sits between the requirement at the current depth and the requirement after the redemption is funded (computable from the commit-time state), keeping no free settlement.
- The fill tick is published at `h`. At `h + tau`:
  - if the price moved for the open, execute it: it fills at the `h` tick;
  - otherwise, call `settleLpEpoch` and then execute: the open fails `PlannerRejected` and the margin comes back.
- Near the skew cap, no band sizing is needed: the redemption pushes the open over `0.4 x depth` with any margin (`SKEW_TOO_HIGH`).

The pre-audit reported this impact as SC-M4 ("Risk Free Option in 60Sec Oracle Window", fixed). Its third component cancelled the losing order through pool depth: "depositing to the junior tranche, then withdrawing the LP after determining the profitable direction ... Executing the losing order then triggers SOLVENCY_EXCEEDED due to reduced vault depth". The fix moved withdrawals to epoch settlement, but it is incomplete. The settlement call is permissionless and unordered against order execution, so the same depth lever still cancels the order at every boundary.

Contradicted text: [SECURITY.md L658-L659](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L658-L659) (quoted above).

## Impact Details
**Low.** The stated security property does not hold, and LPs pay the option value.

PoC: Foundry, unmodified contracts, public entry points only, release parameters (fee 4 bp, VPI 1%, IM 0.2%, MM 0.1%, skew cap 0.4, open bounty 1 bp in [0.01, 0.20] USDG, 10M Junior + 40M Senior, six release feeds). Price path: the real Pyth six-feed basket around the 14 hourly boundaries from 5 Oct 2026 12:00 UTC to 6 Oct 01:00 UTC, interpolated per second between on-chain samples. An honest account holds a 3M-token SHORT.

One open (5 Oct 2026, 14:00 UTC boundary; an ordinary 2M Senior redemption has matured; a 5M-token SHORT of about 4.8M USDG is committed at 13:59:59 with 21,507.90 USDG, inside a 211.03 USDG band):
- open executed first: `Executed`;
- epoch settled first (pool depth 49,999,833.88 -> 47,999,831.33 USDG): `Failed / PlannerRejected` (`INSUFFICIENT_INITIAL_MARGIN`), cost 0;
- with the trader's own 1M Senior redemption instead: band 103.76 USDG, same two outcomes;
- near the cap (15M-token honest SHORT; the trader's 5,811,200-token SHORT at 99.9% of the cap): `Executed` vs `SKEW_TOO_HIGH`, with any margin.

Fourteen real boundaries, with a matured 500k Senior redemption at each. The trader would open the 4.8M SHORT anyway, keeps the fills the price moved toward, and cancels the rest. The control makes the same trades with binding orders committed at the decision time. The honest order keeper uses the measured reference-keeper delays (0-8 s after the fill tick), and the honest LP keeper polls every 15 s.

| Run (14 hours) | Exercised / cancelled / lost to keepers | Trader vs control | LPs vs control |
|---|---|---|---|
| Decide 3 s after the fill tick | 4 / 2 / 8 | +204.39 USDG | -205.11 USDG |
| Commit timed after the order keeper's poll, decide at 6 s | 7 / 1 / 6 | +721.50 USDG | -722.41 USDG |

- An unused attempt costs nothing when the trader executes its own order, and at most the 0.20 USDG bounty when an honest keeper executes it after the epoch settled.
- The option is available at every hourly boundary with a matured redemption, and the trader can supply that redemption itself.

## Recommendation
Do not let a call the trader can order decide a queued open's fate. Price the open's VPI and skew checks with the depth recorded at commitment, or the larger of that and the execution-time depth. Alternatively, have `settleLpEpoch` first execute the queued orders whose fill tick precedes the boundary.

## References
- [OrderRouterExecutionSidecar.sol L953](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L953), [L564-L576](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L564-L576), [L834-L852](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L834-L852)
- [OpenAccountingLib.sol L127-L129](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OpenAccountingLib.sol#L127-L129)
- [CfdEnginePlanLib.sol L112-L128](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L112-L128), [L426-L437](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L426-L437), [L446-L480](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L446-L480)
- [HousePool.sol L856-L862](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L856-L862), [L1004-L1006](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1004-L1006)
- [OrderRouter.sol L479-L484](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L479-L484); [OrderLiquidationBatchLogic.sol L430-L446](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L430-L446), [L549-L589](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L549-L589)
- [PletherOracle.sol L610-L622](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L622)
- [SECURITY.md L658-L659](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L658-L659)
- Pre-audit SC-M4: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf

## Proof of Concept

## Proof of Concept

The full Foundry test and its run output are in this gist: https://gist.github.com/tinnohofficial/806105e22152b939e7d6f93655a3f0db
- `LpSettlementDepthSwitch.t.sol`: 7 tests
- `LpSettlementDepthSwitch.output.txt`: the output of the suite

The test uses the repository's own harness (`BasePerpTest`), its Pyth mock and the six release basket feeds. It calls public entry points only (`commitOrder`, `executeOrder`, `settleLpEpoch`, the TrancheVault request functions), and no in-scope contract is modified.

### How to run
1. Check out `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, and install the dependencies as in the repository README.
2. Save the gist's test file as `packages/perps/test/poc/LpSettlementDepthSwitch.t.sol`.
3. From the repository root, run:

```
forge test --root packages/perps --match-path test/poc/LpSettlementDepthSwitch.t.sol -vv
```

Result: 7/7 pass, and repeated runs give identical output.

### Step by step: one open, two outcomes (`test_DepthSwitch_OneOpenTwoOutcomes`)
1. **Setup:** release parameters, a 10M Junior and 40M Senior pool, and an honest 3M-token SHORT. An ordinary LP's 2M Senior redemption matures at the 5 Oct 2026 14:00 UTC boundary.
2. **Commit:** at 13:59:59 the trader commits a 5M-token SHORT (about 4.8M USDG) with 21,507.90 USDG of committed margin. That is inside the 211.03 USDG band where depth decides the initial-margin check.
3. **Order executed first:** the open ends `Executed` ("open executed first: status / reason 2 1").
4. **Epoch settled first:** the settlement funds 2,000,002.55 USDG, and pool depth drops from 49,999,833.88 to 47,999,831.33 USDG. The same open then ends `Failed / PlannerRejected` ("LP epoch settled first: status / reason 3 7", "receipt failure code (6 = INSUFFICIENT_INITIAL_MARGIN) 6"). The trader's cost is 0.

### Variants
- **`test_DepthSwitch_OwnRedemption`:** the trader's own 1M Senior redemption gives a 103.76 USDG band, with the same two outcomes.
- **`test_DepthSwitch_SkewCap`:** near the skew cap the open is `Executed` if run first. With the epoch settled first it fails `SKEW_TOO_HIGH` (receipt code 4), with any margin.

### Real boundaries (`test_RealBoundaries_*`, 14 hourly boundaries; trader net / LP net, USDG atoms)

| Run | Attack | Control |
|---|---|---|
| Measured keepers, decide at 3 s | -15332173441 / 7632748221 | -15536562753 / 7837855673 |
| Timed commit, decide at 6 s | -26456504161 / 12984966821 | -27178003809 / 13707378169 |

Against the control, that is +204.39 USDG for the trader and -205.11 USDG for the LPs in the first run, and +721.50 / -722.41 USDG in the second.
