# \#98727 [SC-Low] Collected execution fee is reported fully recovered but its treasury delivery is cash-capped: the residual is credited to no party and survives the netting fix — a profit close at low pool cash strands the fee in pool custody

**Submitted on Nov 9th 2026 at 21:05:42 UTC by @Matgothmog for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98727
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

# Description

## Brief

On a profit close, the execution fee is collected by withholding the trader's price gain and the
receipt reports it fully recovered (`ActionChargeSettled`: assessed == recovered, waived == 0).
But the fee's actual delivery to the treasury (`_settleProtocolFeeTopUp`) is capped by free pool
cash, while the LP-revenue booking subtracts the **full** fee from the retained revenue. When free
cash is short — an ordinary state after LP redemptions — the undeliverable residual is credited to
**no party at all**: not the treasury, not LP revenue, not `unassignedAssets`, and no event reports
it. The receipts claim delivery that never happened, and the stranded amount silently blends into
pool custody.

This is a distinct root cause from the earlier-reported net-rebate fee attribution issue: it occurs
on ordinary **profit** closes with `vpiFactor = 0` and is **not addressed by that report's fix**
(recognizing the gross fee in the netting branches does not deliver it when the top-up is cash-
capped). It is also the fix-incomplete counterpart of the MED-6 top-up cap, which bounded the
payment but left the residual unattributed.

## Vulnerability Details

All references are against the in-scope code at commit `a82d4fef`.

1. **Collection**: on a close, the fee is assessed on closed notional (`CloseAccountingLib.sol:121`)
   and collected by withholding it from the trader's settlement —
   `_settleCloseActionCharge` → `clearinghouse.consumeActionCharge(..., host.protocolTreasury(),
   delta.actionProtocolFeeCreditedAssets)` credits the collected part to the treasury account, and
   the withheld-but-not-collected remainder is supposed to be delivered by the top-up.
2. **Delivery is capped**: `_settleProtocolFeeTopUp` (`CfdEngineSettlementSidecar.sol:905-926`)
   computes `poolFundedAssets = amountAssets − clearinghouseCreditedAssets` and pays it from
   `pool.getSettlementFreeAssets()` — **capped by current free cash**. Whatever free cash cannot
   cover is silently dropped: the function returns without recording anything.
3. **Revenue booking subtracts the full fee anyway**: `_recordRetainedCloseRevenue(
   delta.actionChargeWithheldAssets − feeWithheldAssets)` (`CfdEngineSettlementSidecar.sol:615-616`)
   with `feeWithheldAssets = delta.executionFeeAssets − protocolFeeCreditedAssets` (the **full**
   recognized fee) — so the LP-revenue ledger excludes the residual too.
4. **Result**: the residual sits in pool custody without attribution to any ownership bucket. The
   program's accounting rules require "value with no valid claimant path" to sit in explicit
   `unassignedAssets` (`ACCOUNTING_SPEC.md:315`, `:357` — "a new depositor cannot acquire it simply
   by being first to deposit"), while `ACCOUNTING_SPEC.md:782` says fee amounts that cannot be
   cash-credited are "not recorded as a protocol fee receivable". The implementation satisfies
   neither: the value is neither quarantined as `unassignedAssets` nor recorded anywhere — it is
   silently absorbed, and the receipt says the opposite.

**Measured (PoC, production kernel, real state transitions):** LP redeem to the withdrawal firewall
floor (free pool cash = 0) → SHORT close of $4.0M notional at a +$2.0M price gain:
`preview.executionFeeAssets = 2,400,000,000` (recognized gross), treasury credited **0**,
stranded residual **2,400,000,000**, `ActionChargeSettled(assessed == recovered, waived == 0)`.
Control run with ample free cash: the fee is delivered in full (treasury credited 2,400,000,000) —
isolating the top-up cash cap as the sole cause.

## Impact

Impact category: **"Contract fails to deliver promised returns, but doesn't lose value"**.
The treasury is systematically underpaid the collected execution fee whenever free pool cash is
below the top-up need at close time — and the receipt reports the fee as fully recovered while it
was delivered to nobody. The event-backed terminal accounting ("complete terminal economics are
event-backed") therefore misstates the outcome on every affected close, and the unattributed value
violates the program's own claimant-continuity rule. No user funds are taken; the harm is the
misattributed collected charge and the false receipt.

Scale: bounded per close by the fee (4 bps of closed notional at default configuration — $2,400 on
the PoC's $4.0M close) and recurring on every close that lands while free cash is low. The same
reporting also mislabels genuinely recovered frozen spread as waived in the net-rebate family
(exercised by `WavePComposition.t.sol::test_WP3_FrozenClose_ClaimNet_ClampedVpi_WaivedSpread_Conservation`,
4/4 PASS), indicating the receipt attribution needs a systematic fix.

## Recommended Mitigation

When the top-up cannot deliver the full recognized fee, route the residual to an explicit
ownership bucket instead of dropping it: credit it to `unassignedAssets` (satisfying the
claimant-continuity rule) or keep it a recorded treasury receivable until cash is available. And
make the receipts truthful: `ActionChargeSettled` must report recovered = actually delivered, with
the shortfall reported separately — the same treatment should cover the frozen-spread
recovered/waived split so that "waived" only ever denotes genuinely uncollectible value.


## Link to Proof of Concept

https://gist.github.com/Matgothmog/7447df6aa861c7a2943572af5e613d92

## Proof of Concept

# Proof of Concept

Foundry tests against the frozen commit `a82d4fef` (Forge 1.5.1, solc 0.8.35 via-IR).
File: `poc/WaveXFeeTopUpResidual.t.sol` (drop into `packages/perps/test/perps/regression/` and run
`forge test --root packages/perps --match-contract WaveXFeeTopUpResidual -vv`).
Full run output: `poc/RUN_OUTPUT.txt` — **2/2 PASS**.

All state is driven through production entrypoints (LP redemption through the async vault cycle to
the withdrawal firewall floor; the close through `engine.processOrderTyped`); no storage writes to
protocol state.

1. **`test_FeeTopUpCashCapped_ResidualCreditedToNobody`** (the defect):
   LP redeems to the firewall floor so free pool cash = 0, then a SHORT close of $4.0M notional at a
   +$2.0M price gain executes:
   ```
   fee recognized (preview):       2,400,000,000
   treasury credited:              0
   stranded residual (attributed to nobody): 2,400,000,000
   ActionChargeSettled: assessed == recovered, waived == 0   (receipt says fully delivered)
   ```
   The fee is collected by withholding it from the trader's payout, `_settleProtocolFeeTopUp`'s
   free-cash cap drops the delivery, `_recordRetainedCloseRevenue` subtracts the full fee from LP
   revenue — and the residual is left unattributed in pool custody.

2. **`test_Control_AmpleCash_DeliversFeeInFull`** (isolation): identical close with ample free cash
   credits the treasury 2,400,000,000 in full — the top-up cash cap is the sole cause.

Deterministic; no forks; no privileged actors. The trigger state (free cash below the top-up need)
is ordinary and permissionlessly reachable (a single LP redemption to the documented firewall floor
produces it), and the affected close is an ordinary profit close with `vpiFactor = 0`.
