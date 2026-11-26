# \#99817 [SC-Medium] Oracle-frozen liquidation pricing subtracts the SHORT adverse shift from an already-capped mark, letting any keeper liquidate healthy short positions during weekend/holiday market closures

**Submitted on Nov 15th 2026 at 12:03:05 UTC by @bugtester for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99817
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief / Intro
The oracle's liquidation-pricing policy is: shift the neutral basket by the adverse confidence amount, then clamp to
`CAP_PRICE`. On the live-Pyth branch the code does exactly that. While the oracle is frozen (FX weekends and governance
holiday overrides), however, the liquidation basket is `_frozenBasket`, whose price is the Engine's cached mark — a
value that is already clamped to `CAP_PRICE`. The SHORT shift is then subtracted from that capped value, producing a
liquidation price of `CAP − shift` instead of `CAP`. A SHORT that is healthy at the policy-correct price becomes
liquidatable, and any permissionless keeper can liquidate it during the frozen window, taking half of its liquidation
charge.

## Vulnerability Details
1. **The frozen basket is the capped Engine mark.** While `isOracleFrozen()`, `_resolveCurrentBasket`
   (PletherOracle.sol:749-756) returns `_frozenBasket` (PletherOracle.sol:729-744):
   ```solidity
   basket.price = engine.lastMarkPrice();                                  // already clamped to CAP_PRICE
   basket.confidence = basket.price * basketMaxConfidenceRatioBps / 10_000;
   ```
   `CfdEngine.updateMarkPrice` clamps every mark to `CAP_PRICE`, and the uncapped basket is never stored.
2. **The shift is applied to that capped value.** `updateLiquidationPrice` (PletherOracle.sol:334-342) and
   `updateLiquidationBatchPrice` (PletherOracle.sol:308-317) take `basket.price` as the "raw" price and apply the SHORT
   shift via `_directionalLiquidationPriceFromShift` (PletherOracle.sol:966-972): `price - shift`. The final
   `_clampToCap` is a no-op.
3. **Result.** When the market froze with the basket at or above the cap, the SHORT liquidation price is `CAP − shift`,
   whereas the live branch for the same market returns `min(raw − shift, CAP) = CAP`. Single (`executeLiquidation`) and
   batch (`executeLiquidationBatch`) liquidation are both affected.

## Impact Details
- A SHORT that is healthy at the policy-correct price (CAP) but within `shift` of the frozen-market margin threshold
  becomes liquidatable during the frozen window and is force-closed by any permissionless keeper.
- The victim loses the liquidation charge (`bountyBps` 10 = 0.1% of notional; 50% credited to the keeper, 50% to LPs),
  is settled `lots × shift` below CAP, and loses the position. In the PoC, compared with the correct (not liquidated)
  outcome, the victim is **23.90 USDG worse off** and the liquidating keeper **gains 9.95 USDG**, on a 10,000-token
  (~20k USDG notional) position that is healthy at the policy-correct price. Loss scales linearly with position size.
- The victim cannot defend: in the frozen window opens are close-only and no new oracle data can move the mark. The
  frozen window recurs every weekend (~48h) and on configured holidays.

## References
- packages/perps/src/PletherOracle.sol:729-744 (`_frozenBasket`), :749-756 (`_resolveCurrentBasket`), :308-317
  (`updateLiquidationBatchPrice`), :334-342 (`updateLiquidationPrice`), :966-972 (`_directionalLiquidationPriceFromShift`)
- packages/perps/src/CfdEngine.sol `updateMarkPrice` (mark clamped to CAP_PRICE)
- packages/perps/ORACLE_SYNCHRONIZATION.md:74 (frozen liquidations apply a deterministic adverse shift)


## Link to Proof of Concept

https://gist.github.com/piyushshukla599/45734d3daf9bf37c18bb80dffd186e8d

## Proof of Concept

## Proof of Concept

### Attached PoC files
| File | Place at (in the repo) |
|---|---|
| `E:\plether-audit\submission\poc\test\perps\poc\FrozenShortLiquidationBelowCap.t.sol` | `packages/perps/test/perps/poc/FrozenShortLiquidationBelowCap.t.sol` |

- Single self-contained file built on the repo's own `packages/perps/test/perps/BasePerpTest.sol`; no in-scope contract
  is modified. Only Pyth (repo MockPyth) and USDG (repo MockSettlementToken) are mocked.
- The test overrides `_riskParams()` with the DEPLOYMENT.md release risk parameters (maint 10 bps, init 20 bps,
  vpiFactor 0.01e18, bountyBps 10, keeperShareBps 5000).

### Run
```
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
# copy the file to the path above, then:
forge test --root packages/perps --match-path 'test/perps/poc/FrozenShortLiquidationBelowCap.t.sol' -vv
```
Tip: the full perps test tree compiles slowly under via-IR. For a fast run, create a folder containing
`perps/BasePerpTest.sol`, the `utils/` helpers it imports (LegacyOrderRouterHarness, MockSettlementToken,
OrderRouterDebugLens, RecordedOrderReceipts, LegacyOrderRequestEncoding) and the PoC file at the same relative
paths, then run with `FOUNDRY_TEST=<that folder>`.

### Expected output (verified at a82d4fe)
```
[PASS] test_FrozenFalseLiquidationOfHealthyShort() (gas: 59992249)
  victim margin (USDG atoms): 529836098
  frozen SHORT liquidation price: 199960000
  victim size before: 10000000000000000000000
  victim size after: 0
  victim equity, baseline (healthy, valued at CAP): 1619900001
  victim equity, actual (after false liquidation): 1596000001
  VICTIM LOSS vs baseline (USDG atoms): 23900000
  KEEPER GAIN (USDG atoms): 9950000
[PASS] test_FrozenShortLiquidationPriceBelowCap() (gas: 2274564)
  live SHORT liquidation price: 200000000      <- live branch returns CAP
  frozen SHORT liquidation price: 199960000    <- frozen branch returns CAP - shift
  frozen neutral mark: 200000000
  frozen batch shortPrice: 199960000
Suite result: ok. 2 passed; 0 failed
```
(The test harness oracle uses its default adverse multiplier; at the release value of 2,500 bps the shift is larger,
widening the affected margin band.)

### Steps performed by `test_FrozenFalseLiquidationOfHealthyShort`
1. Friday 20:00 UTC (live): `router.updateMarkPrice` with a raw basket of 2.1 (> CAP 2.0); the Engine mark is cached at
   CAP.
2. The victim opens a 10,000-token SHORT at 1.99. Its margin is binary-searched to the smallest value for which
   `CfdEngineLens.isLiquidatableAt(victim, CAP, depth) == false` during the frozen window — the victim is healthy at the
   policy-correct price.
3. Saturday 12:00 UTC (oracle frozen): `updateLiquidationPrice` returns 1.9996 (< CAP); the victim is liquidatable at that
   price.
4. Accrued carry is realized first (`withdrawMargin(0)`), so baseline and actual outcomes bear identical carry.
5. Baseline (correct behaviour, not liquidated): victim equity = custody + trader claim + exact SHORT PnL at CAP
   = 1,619.90 USDG.
6. A keeper calls `router.executeLiquidation(victim, [])`: the position goes 10,000 → 0. Actual victim equity =
   1,596.00 USDG.
7. Asserted: **victim loss vs baseline = 23.90 USDG** (19.90 liquidation charge = 0.1% of ~19,900 notional, plus 4.00
   from settling at 1.9996 instead of 2.0000) and **keeper gain = 9.95 USDG**, taken from a position that was healthy at
   the policy-correct price.

`test_FrozenShortLiquidationPriceBelowCap` shows the same market returning CAP on the live branch and CAP − shift on the
frozen branch, for both single and batch liquidation pricing.

---

## Recommendation
In the frozen branch the true raw basket is unknown and was at or above the cached mark. Do not apply the SHORT adverse
shift below CAP when the frozen mark equals CAP (e.g. return CAP for SHORT when `capped == CAP`), or persist the last
uncapped accepted basket price and apply the shift-then-clamp ordering to it in `_frozenBasket`.


