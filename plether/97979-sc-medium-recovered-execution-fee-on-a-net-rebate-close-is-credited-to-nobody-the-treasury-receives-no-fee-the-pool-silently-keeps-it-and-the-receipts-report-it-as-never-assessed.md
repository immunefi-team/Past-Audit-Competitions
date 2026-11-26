# \#97979 [SC-Medium] Recovered execution fee on a net-rebate close is credited to nobody: the treasury receives no fee, the pool silently keeps it, and the receipts report it as never assessed

**Submitted on Nov 4th 2026 at 07:22:01 UTC by @Matgothmog for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97979
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief

On any close whose net action settlement is a **rebate** (the trader's VPI rebate exceeds the sum of execution fee, frozen-close spread and net carry), the planner still assesses the execution fee on the closed notional, and the fee is in fact economically collected from the trader — but it is credited to **neither the protocol treasury nor LP revenue accounting**, and the close receipts report it as never assessed.

The result is a silent, repeatable misrouting of protocol fee revenue into LP equity, plus event-backed economics that contradict what actually happened. A profit close on the identical trade shape credits the treasury correctly; only the net-rebate branch loses the fee.

## Vulnerability Details

All references are against the in-scope code at commit `a82d4fef`.

**1. The fee is always assessed.** `CloseAccountingLib` computes the close fee unconditionally from the closed notional (`libraries/CloseAccountingLib.sol:121`, `state.executionFeeAssets = notionalAssets * executionFeeBps / 10_000`), and `CfdEngineAdmin` rejects any zero fee rate (`CfdEngineAdmin.sol:113`, `executionFeeBps == 0 || > 10_000` reverts), so every close carries a positive fee.

**2. In the rebate branch the fee is economically collected but recognized as zero.** In `_planCloseActionSettlement` (`libraries/CfdEnginePlanLib.sol:1034-1068`) the net action settlement is

```text
actionNet = vpiDelta + (executionFee + frozenSpread + pendingCarry - realizedCarry)
```

When `actionNet < 0`, the code sets only `actionRebateAssets = -actionNet` — i.e. the trader's gross VPI rebate **minus** the fee, spread and carry — and leaves `actionChargeWithheldAssets` and `actionChargeCollectedAssets` at zero. The fee-allocation block then computes `executionFeeEligibleAssets = nonClawbackWithheld + nonVpiReserveCollected = 0`, so `delta.executionFeeAssets` and `delta.actionProtocolFeeCreditedAssets` are both zeroed (`CfdEnginePlanLib.sol:1061-1068`).

**3. Settlement credits nobody.** In `CfdEngineSettlementSidecar.sol:604-616` the LP-revenue computation runs `cashArrivedRevenueAssets = pricePnlPledgeConsumed + actionChargeCollected(0) - protocolFeeCredited(0)` and `_recordRetainedCloseRevenue(actionChargeWithheld(0) - feeWithheld(0)) = 0`, and the protocol-fee top-up `_settleProtocolFeeTopUp(host, delta.executionFeeAssets = 0, ...)` returns immediately (`:905-925`). The pool simply pays out the shrunken rebate (`payOut` + `settleAssets` on `actionRebatePaidAssets`), so the fee-sized difference stays in the pool's accounted assets and accrues to LP equity.

**4. Receipts misreport the outcome.** `ActionChargeSettled` is only emitted when `actionChargeAssessedAssets > 0` (`CfdEngineSettlementSidecar.sol:673-680`) — in the rebate branch it never fires, and `ActionRebateSettled` reports the net rebate that silently has the fee removed from it. For frozen-market closes, `FrozenCloseSpreadSettled` reports the recovered spread as fully **waived**: `_recoveredFrozenSpreadAssets` (`:724-748`) computes recovery from `actionChargeWithheld + actionChargeCollected`, which are both zero in this branch, so the event claims `recovered = 0, waived = spread` even though the spread was in fact collected from the rebate. This contradicts the terminal-receipt contract in `SECURITY.md` ("complete terminal economics are event-backed") and the frozen-spread conservation invariant ("assessed spread equals LP-paid spread plus terminally waived spread").

**5. Spec contradiction.** `INTERNAL_ARCHITECTURE_MAP.md` ("Trading flows" step 5): "Collected carry/spread belong to LPs; protocol fees go to treasury Clearinghouse margin." `ACCOUNTING_SPEC.md`: "Revenue with no eligible share owner is routed to `unassignedAssets`; a new depositor cannot acquire it simply by being first to deposit." Recovery-by-netting is still collection: the trader pays the fee in kind, so the value must be routed to the treasury account, not silently retained as pool equity.

**Concrete demonstration (PoC numbers).** Open LONG 100,000 (18 dec) at 1.0 with 9,000 margin and `vpiFactor = 0.1e18`, then close 50,000 at 0.8 in live mode at the same timestamp (zero carry, no spread):

| Quantity | Value (6-dec USDG) |
|---|---|
| assessed fee (4 bps × $40,000 closed notional) | 16,000,000 |
| gross VPI rebate (`-vpiDeltaAssets`) | 239,401,734 |
| rebate actually paid to trader | 223,401,734 |
| difference | **16,000,000 = exactly the fee** |
| credited to protocol treasury | **0** |
| `preview.executionFeeAssets` reported | **0** |

The contrast is in the project's own test suite: `PayoutModesMatrixTest.test_CloseImmediatePayoutMode` shows a profit close on the same shape crediting the treasury `32e6` ("Price profit less the 4 bps close fee").

## Impact

- The protocol treasury is underpaid the full execution fee on every net-rebate close (skew-reducing closes with a VPI rebate larger than the fee + carry). At the default 4 bps this is 0.04% of closed notional per affected close, recurring and permissionless (any trader closing a skew-reducing position triggers it).
- The fee-sized value silently accretes to LP equity instead of the treasury account, contrary to the documented fee model; in a zero-principal window the same value is captured by later depositors instead of sitting in `unassignedAssets`, contrary to the claimant-continuity rule.
- All receipts and previews for these closes misstate the economics (fee shown as never assessed; frozen spread shown as waived though recovered), breaking the event-backed accounting contract that indexers, monitoring (`SettlementMonitorLens`-style consumers) and the frozen-spread conservation invariant rely on.

## Recommended Mitigation

In `_planCloseActionSettlement`, recognize the fee in the rebate branch too: compute `executionFeeEligibleAssets` from the gross rebate withholding (e.g. credit `min(cs.executionFeeAssets, actionRebateAssets)` to the treasury and treat the trader's payout as `actionRebateAssets - fee`), or alternatively gross-up the rebate and collect the fee separately so the two paths are symmetric. Also fix the receipt math (`_recoveredFrozenSpreadAssets`, `ActionChargeSettled`) to report value that was recovered through rebate netting as recovered rather than waived.


## Link to Proof of Concept

https://gist.github.com/Matgothmog/2fc4e5a9b41ee0a8ee605f3372b41791

## Proof of Concept

# Proof of Concept

Foundry regression test against the frozen commit `a82d4fef` (Forge 1.5.1, solc 0.8.35 via-IR).
File: `poc/NetRebateCloseFeeCredit.t.sol` (drop into `packages/perps/test/perps/regression/` and run
`forge test --root packages/perps --match-contract NetRebateCloseFeeCredit -vv`).
Full run output: `poc/RUN_OUTPUT.txt` — **PASS**.

The test drives a real close through `CfdEngine.processOrderTyped` (the same accounting kernel the
Router uses) on the project's own `BasePerpTest` harness and asserts, with the planner's own values:

1. `preview.executionFeeAssets == 0` while the fee actually charged is `(40_000e6 × 4 bps)/10_000 = 16_000_000`
   — the preview reports no fee.
2. `grossRebate - rebatePaid == 16_000_000` **exactly** — the trader's VPI rebate
   (`-vpiDeltaAssets = 239,401,734`) is paid out as only `223,401,734`; the difference is precisely
   the 4-bps execution fee. The fee is therefore economically collected from the trader.
3. `clearinghouse.balanceAssets(engine.protocolTreasury())` is **unchanged** across the close —
   none of the collected fee reaches the treasury.
4. The pool's USDG outflow equals `payoutToPnlPledge + rebatePaid` — the fee-sized value is retained
   by the pool as equity (no top-up, no revenue booking).
5. No `ActionChargeSettled` event is emitted despite the fee having been assessed and recovered.

Observed values (from the run output):

```
engine.executionFeeBps: 4
preview.executionFeeAssets (recognized?): 0
expectedFee (4bps x 40k): 16000000
preview.vpiDeltaAssets: -239401734
grossRebate (= -vpiDelta): 239401734
rebatePaid (payoutToFreeSettlement): 223401734
payoutToPnlPledge: 10000000000
immediatePayout: 10223401734
```

Contrast (same repository, existing test `PayoutModesMatrixTest.test_CloseImmediatePayoutMode`):
a profit close on the identical trade shape asserts
`clearinghouse.balanceAssets(engine.protocolTreasury()) == treasuryBefore + 32e6` with the comment
"Price profit less the 4 bps close fee is paid immediately" — proving the intended destination of the
fee and that only the `actionNet < 0` branch loses it.

The test is deterministic, needs no fork, and uses only production entrypoints plus the standard
test-harness funding helpers.
