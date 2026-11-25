# \#98536 [SC-Insight] Opens whose execution window crosses the FAD start can be liquidated as soon as FAD begins, with no price move

**Submitted on Nov 8th 2026 at 14:09:13 UTC by @blackgrease for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98536
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Brief/Intro

An open order can be committed with an execution window that runs past the start of the FAD window. If it fills just before FAD starts, it only has to meet the normal margin rule. Seconds later the maintenance requirement jumps to 3% and the same position, at the same price, becomes liquidatable, so the trader pays the liquidation charge on a position they barely held.

## Vulnerability Details

Under normal conditions, the weekly FAD window starts 30 minutes before the Friday FX close and is worked out from the current block time:

```solidity
if (dayOfWeek == 5) {
    uint256 marketBoundary = newYorkMarketBoundary(timestamp, dayOfWeek);
    fadWindow = secondOfDay >= marketBoundary - FAD_LEAD;
    oracleFrozen = secondOfDay >= marketBoundary;
}
```
Code location: [MarketCalendarLib.sol#L38-L41](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarketCalendarLib.sol#L38-L41)

Every order carries an execution window chosen by the trader, up to the router's maximum (60 seconds by default, configurable up to 1 hour). Its execution deadline is the commit time plus that window:

```solidity
if (bounds.executionWindowSeconds == 0 || bounds.executionWindowSeconds > host.maxExecutionWindowSeconds()) {
    revert OrderRouter__InvalidExecutionWindow();
}
```
Code location: [OrderLiquidationBatchLogic.sol#L287-L289](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L287-L289)

```solidity
uint256 deadline = block.timestamp + uint256(request.bounds.executionWindowSeconds);
```
Code location: [OrderLifecycleBook.sol#L287](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L287)

An open is only refused at commit if FAD has already started. **Nothing compares its execution deadline with the next FAD start**. Therefore, an order committed at 20:29:30 with a 60-second window (the default maximum), or at 19:35 with a 1-hour window (the configurable maximum), is accepted even though it may fill moments before FAD. The trader doesn't choose when inside the window the order fills; keepers and the queue decide that.

As execution only holds opens back from the FAD start, the order can fill at any point up to 20:29:59. When it fills, its health check uses whichever maintenance rate applies at that second, which is the normal rate:

```solidity
riskState = PositionRiskAccountingLib.buildExactPriceRiskState(
    projectedPosition,
    delta.newPosEntryCostAssetAtoms,
    delta.price,
    snap.capPrice,
    reachableCollateralAssets,
    snap.isFadWindow ? snap.riskParams.fadMarginBps : snap.riskParams.maintMarginBps
);
```
Code location: [CfdEnginePlanLib.sol#L678-L686](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L678-L686)

The open is accepted if equity beats the initial margin and the position isn't liquidatable at that rate:

```solidity
if (
    postOpenRiskState.liquidatable
        || postOpenRiskState.equityAssets <= int256(openState.initialMarginRequirementAssets)
) {
    delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.INSUFFICIENT_INITIAL_MARGIN;
    return delta;
}
```
Code location: [CfdEnginePlanLib.sol#L431-L437](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L431-L437)

From the first second of FAD, liquidation uses the FAD rate for every position, including one opened a second earlier:

```solidity
uint256 maintMarginBps = snap.isFadWindow ? snap.riskParams.fadMarginBps : snap.riskParams.maintMarginBps;
```
Code location: [CfdEnginePlanLib.sol#L1450](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1450)

With the default rates (0.2% initial, 3% FAD), any open with equity between 0.2% and 3% of its size whose window crosses the FAD start can fill just before FAD and be liquidatable once it starts.

A trader opening from a fresh account can't prepare for this either. `addMargin` needs an existing position, so there is nothing to top up before the order fills:

```solidity
StoredPosition storage pos = _positions[account];
if (pos.lots == 0) {
    revert CfdEngine__NoOpenPosition();
}
```
Code location: [CfdEngine.sol#L512-L515](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L512-L515)

After the fill, the trader has only the seconds left before FAD to add margin, and has to beat liquidators to it.

### Example scenario

Using the test suite's rates (1.5% initial, 1% maintenance, 3% FAD) on a Friday, when FAD starts at 20:30:00 UTC:

1. At 20:29:30 Alice, with no open position, commits a $10,000 LONG with $250 margin and a 40-second execution window.
2. Her order stays fillable until 20:30:10, ten seconds past the FAD start, and the commit is still accepted.
3. A keeper executes the order at 20:29:59, 1 second before FAD starts. Her equity is $236, above the $150 initial and $100 maintenance requirements, so the open succeeds.
4. At 20:30:00 FAD starts. The price hasn't moved, but the requirement is now $300 (3%), so her $236 position is liquidatable.
5. A keeper liquidates her in the same second. Her position is closed and she has paid $14.20 (opening fee, keeper bounty and the $10 liquidation charge) for a position she held for one second.

## Impact

**Security best practices:** The protocol accepts positions that its own fixed, on-chain FAD schedule will put below the liquidation line seconds later. This isn't a market risk the trader could have priced; the loss comes from where inside the execution window the fill lands, which the trader doesn't control. The trader pays the liquidation charge (0.1% of size, so $1,000 on a $1M position) plus the opening costs, and loses the position they asked for.

On the adversarial side, malicious keepers also have a reason to wait and fill such orders right before FAD as they collect the execution bounty and then the liquidation bounty a second later. However, this only works when no honest keeper executes the order first.

## Recommendations

At commit, if an open's execution deadline is at or after the next FAD start, require its margin to meet the FAD rate, or reject it. A simpler option is to make opens close-only for a lead period at least as long as the maximum execution window - the way the holiday runway (`fadRunwaySeconds`) already works for override days.

## Link to Proof of Concept

https://gist.github.com/blackgrease/79de1a20bfb1572c7369db6cf30425ae

## Proof of Concept

The PoC exists in the private gist link and reproduces the issue - https://gist.github.com/blackgrease/79de1a20bfb1572c7369db6cf30425ae

**Understanding the PoC**

1. A fresh account commits a $10,000 LONG with $250 margin at 20:29:30 on a Friday, with a 40-second window. The order stays fillable until 20:30:10, past the FAD start. `addMargin` reverts because there is no position yet.
2. A keeper executes it at 20:29:59. It passes the normal margin check.
3. At 20:30:00 FAD starts. Same price, but the position is now liquidatable.
4. The keeper liquidates it. The position is closed and Alice has lost $14.20 with no price move.
5. Control: the same order reaching the keeper at 20:30:00, still inside its window, is held back by FAD. No position opens and Alice's balance is unchanged.

**Instructions to run**

1. Place code in: `packages/perps/test/perps/spec/trader/PoC_FadBoundaryOpenLiquidation.t.sol`
2. Run with: `forge test --root packages/perps --mc PoC_FadBoundaryOpenLiquidation -vv`

**Console Output:**

```bash
Ran 2 tests for test/perps/spec/trader/PoC_FadBoundaryOpenLiquidation.t.sol:PoC_FadBoundaryOpenLiquidation
[PASS] testControl_SameOrderAtFadStartDoesNotOpen() (gas: 2282581)
Logs:
  [CONTROL] Same order at 20:30:00: held back by FAD, no position, Alice's balance unchanged at $1000.00

[PASS] testPoC_OpenJustBeforeFadIsLiquidatableOneSecondLater() (gas: 3436556)
Logs:
  1. 20:29:30 Alice commits a $10,000 LONG, $250 margin, 40s window (fillable until 20:30:10); addMargin reverts (no position)
  2. 20:29:59 Order executes. Equity $236.00 vs required $100.00 (1%)
  3. 20:30:00 FAD starts, same price. Equity $236.00 vs required $300.00 (3%) -> liquidatable
  4. 20:30:00 Keeper liquidates Alice. Position closed, she lost $14.20 with no price move

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 41.53ms (23.93ms CPU time)

Ran 1 test suite in 116.20ms (41.53ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

