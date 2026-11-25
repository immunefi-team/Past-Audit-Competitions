# \#98936 [SC-Medium] Heavy-side close nets the execution fee against the VPI rebate: the trader is charged the fee, the treasury receives 0 and LPs keep it, while an open with the same rebate pays the treasury in full

**Submitted on Nov 11th 2026 at 02:22:10 UTC by @dxhod for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98936
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

On a voluntary close that reduces skew (a close on the heavy side of the book), `CfdEnginePlanLib._planCloseActionSettlement` nets the 4 bp execution fee against the close's VPI rebate and then treats none of the fee as owed to the treasury. The trader is still charged the fee, because the rebate paid out is reduced by it, but the treasury is credited 0 and no top-up is made. The fee stays in the HousePool and accrues to LPs. An open with the same rebate is settled differently: the fee stays owed and is topped up to the treasury from free pool cash. With release parameters, every heavy-side close credits the treasury nothing once the skew midpoint reaches 4% of pool depth; below that, the treasury loses the part of the fee the rebate covers. No attacker, admin action or special state is needed, and free pool cash is ample (701,699 USDG in the PoC).

## Vulnerability Details

The close planner folds the VPI delta, the execution fee, the frozen spread and carry into one signed amount. When the rebate is larger than the fee, the result is a net rebate with the fee already subtracted.

`CfdEnginePlanLib.sol:1041-1061`:

```solidity
        int256 actionNetAssets = cs.vpiDeltaAssets
            + int256(
                cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets
            );
        ...
        } else if (actionNetAssets < 0) {
            delta.actionRebateAssets = uint256(-actionNetAssets);
        }
```

The fee routed to the treasury is then capped by what was withheld from a price gain or collected in cash. On a rebate close both are 0, so the fee is dropped.

`CfdEnginePlanLib.sol:1066-1068`:

```solidity
        uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;
        delta.executionFeeAssets =
            cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets;
```

Because `delta.executionFeeAssets` is now 0, the top-up at `CfdEnginePlanLib.sol:984-987` (`feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets`) is also 0, regardless of free pool cash. The sidecar pays the trader the rebate minus the fee from the pool and credits nothing to the treasury.

The open path handles the same netting differently. In `CfdEnginePlanLib.sol:718-749` the fee stays at its full amount when the trade cost is below it, and the shortfall is topped up from free pool cash; `CfdEngineSettlementSidecar._settleProtocolFeeTopUp` (`:905-926`) pays it to the treasury. Inside the close function itself, a fee withheld from a price gain also stays eligible (`nonClawbackWithheldAssets`) and is topped up. That case is economically the same as a rebate offset: the pool owes the trader an amount and pays it net of the fee. Only the rebate offset loses the fee.

The documentation assigns the fee to the treasury. `WHITEPAPER.md:411-413` says "the protocol execution fee belongs to treasury rather than LP assets", and the fee table at `WHITEPAPER.md:810` says "Treasury, only when cash credited". `ACCOUNTING_SPEC.md:782` defines the uncredited case as a fee "that cannot be cash-credited under this priority", meaning free cash is not available after trader claims and payouts, and `:1118` lists free-cash-funded top-ups as treasury margin. Here free cash is available, so under the spec's own rule the fee should be credited. No document names LPs as a recipient of the execution fee.

This is not the cash-limited partial funding covered by Cecuro MED-6 and LOW-26. Those concern top-ups that cannot be funded; here free pool cash is 701,699 USDG and the planner never requests the top-up.

History (read from code, not executed): before refactor `566bea19b` (PR #62), the close path netted the VPI and the fee into one settlement amount and, on the gain branch, topped up the full fee from pool cash (`CfdEnginePlanLib.sol` at `863b691ef`, lines 680-691). A rebate-netted close then credited the treasury the same way an open does. The `WHITEPAPER.md:810` line predates that refactor. Earlier, commit `ddc5043f0c` ("Fix close fee offset accounting") asserted that a profit-offset fee "should be marked for pool top-up".

Suggested fix: on a close with a net action rebate, keep the fee eligible and route `executionFeeAssets - credited` through the existing `_freePoolCashForProtocolFee` top-up, as the open path and the gain-withheld close already do.

## Impact Details

Severity: Low — Contract fails to deliver promised returns, but doesn't lose value.

The protocol treasury, which the program lists as a fee recipient, does not receive the execution fee on heavy-side closes. The trader still pays it and the LPs receive it instead, so total value is conserved and no user loses funds.

PoC results at commit a82d4fe with release parameters (vpiFactor 0.01e18, maxSkewRatio 0.4, execution fee 4 bps), pool depth about 1,002,449 USDG, price constant at 1.0:

- A 100,000 notional LONG close at 300,000 LONG skew: fee assessed 40 USDG, net rebate paid 209.39 USDG (rebate 249.39 minus the fee), treasury credited 0, top-up 0.
- Compared with the same close at a fee rate of 0, LP cash and Junior NAV are 40.000000 USDG higher and the trader receives 40.000000 USDG less. The fee moves from the trader to the LPs.
- A 100,000 SHORT open with the same skew move and the same 249.39 USDG rebate credits the treasury 40.000000 USDG from pool cash.
- The same position closed in other book states: with zero VPI or a VPI charge, the treasury gets the full 40 USDG; with a 19.96 USDG rebate, it gets 20.04 USDG (fee minus rebate).
- The treasury credit reaches 0 when (background skew + close size / 2) / depth reaches 4.00%, measured by bisection for 1k, 10k and flat-book closes. In a flat book, any close of 80,200 USDG or more (8% of depth) credits 0. All 39 sampled heavy-side closes credited less than the full fee.

The amount is up to 4 bps of heavy-side close notional, which is 400 USDG per 1,000,000 USDG closed above the 4% threshold (derived from the measured rule). How often a live book sits above 4% skew is not measured, since the release has no trading history.

Known issues: the behaviour is not in the three listed audits, `SECURITY.md:687-700` or `SECURITY.md:1328-1457`. The VPI limitations there cover the lifetime clamp, depth and liquidation only.

## References

- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1036-1074` — `_planCloseActionSettlement`: fee netted against the rebate, eligibility set to 0
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:984-987` — close fee top-up computed from the already-zero fee
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:718-749` — open path keeps the fee and tops it up
- `packages/perps/src/CfdEngineSettlementSidecar.sol:442-453`, `:905-926` — open rebate payout and `_settleProtocolFeeTopUp`
- `packages/perps/WHITEPAPER.md:411-413`, `:810`
- `packages/perps/ACCOUNTING_SPEC.md:782`, `:1118`
- Commits `566bea19b`, `863b691ef`, `ddc5043f0c` (history)
- Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`


## Link to Proof of Concept

https://gist.github.com/dxhod/c3f3d2d0ac9404ff561a12fa081154aa

## Proof of Concept

Foundry test on commit a82d4fe using the repository's `BasePerpTest` harness with release parameters, which are asserted at the start of every test. Only USDG and Pyth are mocked, and every trade is `OrderRouter.commitOrder` by the trader followed by `executeOrder` by an unrelated keeper.

### Steps

1. Apply the release Router and HousePool parameters through the owner's timelocked propose/finalize before any position exists. Junior holds 1,000,000 USDG.
2. ANCHOR opens LONG 200,000 and HEAVY opens LONG 100,000 (skew 300,000 LONG). HEAVY's open credits the treasury 40 USDG.
3. HEAVY closes LONG 100,000. Record the assessed fee, the `ActionRebateSettled` event and the treasury, pool, Junior NAV and trader deltas.
4. From the same snapshot, execute instead a SHORT 100,000 open with the same skew move and record the same values.
5. Control, for measurement only: from the same snapshot, repeat the close with `executionFeeBps` set to 0 through `stdstore` to show where the fee value ends up.
6. Test 2 closes the same position after different book moves (zero VPI, VPI charge, rebate below the fee, rebate above the fee). Tests 3a-3f sweep background skew and close size and bisect the threshold.

### Command

Place the file at `packages/perps/test/sca/H56R.t.sol`, then from the repository root:

```sh
forge test --root packages/perps --match-path test/sca/H56R.t.sol -vvv
```

### Result

`8 passed`: `test_H56R_1_heavyClose_feeAssessedNotCredited_LPsKeepIt_vsOpenTopUp`, `test_H56R_2_controls_rebateZeroOrBelowFee_creditsTreasury`, `test_H56R_3a_sweep_close1k`, `test_H56R_3b_sweep_close10k`, `test_H56R_3c_sweep_close100k`, `test_H56R_3d_threshold_close1k`, `test_H56R_3e_threshold_close10k`, `test_H56R_3f_threshold_flatBook_closeSize`.

- Close: fee assessed 40.000000, net rebate paid 209.389232 (rebate 249.389232 minus the fee), treasury +0.
- Open with the same rebate: treasury +40.000000.
- Against the fee-0 control: LP cash +40.000000, Junior NAV +40.000000, trader -40.000000, treasury unchanged.
- Threshold: the credit is 0 from (background skew + close / 2) / depth = 4.00%.

### Key assertions

- The `ActionRebateSettled` amount equals the rebate minus the fee minus carry, and the treasury delta on the close is 0. No storage write is involved.
- The treasury delta on the open with the same rebate equals the fee. No storage write is involved.
- Real close minus the fee-0 control: LP cash = +fee, Junior NAV = +fee (within 2 atoms), Senior NAV unchanged, trader = -fee.
- Pool, treasury, trader and keeper deltas sum to 0 on every order.
- Test 2: full credit with zero VPI or a VPI charge; credit = fee - rebate when 0 < rebate < fee; credit 0 when rebate > fee.
- Sweep: on every row the credit equals clamp(fee + carry - rebate, 0, fee); it is 0 above 4.01% and positive below 3.99%.

`vm.prank(engine)` is used only to read the view `buildRawSnapshot`. The `stdstore` write exists only in the control branch, after reverting to the snapshot.

