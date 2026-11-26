# \#97793 [SC-Insight] `PletherOracle._invertPythPrice` rounds half-up at 1e-26 and then floors to 1e-8, so inverted basket components are always truncated

**Submitted on Nov 3rd 2026 at 08:37:54 UTC by @W3B_SR_0xki for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97793
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

**Target:** `packages/perps/src` tree at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (PletherOracle.sol)
**Impact:** Insight — Code optimizations and enhancements

## Description

At release, four of the six basket components are inverted Pyth feeds: USD/JPY, USD/CAD, USD/SEK and USD/CHF (DEPLOYMENT.md:168). Their normalized price comes from `PletherOracle.sol:1008-1019`:

```solidity
uint256 scaledPrecision = 10 ** uint256(uint32(26 - expo));
uint256 scaledInverse = (scaledPrecision + (positivePrice / 2)) / positivePrice;   // half-up at 1e-26
return scaledInverse / 1e18;                                                        // floor to 1e-8
```

The `+ positivePrice / 2` term rounds half-up at 26-decimal precision. The following `/ 1e18` then floors to 8 decimals, which discards that rounding. The function therefore always truncates the inverse toward zero.

The same function computes the inverted confidence endpoint (`upperNorm`, PletherOracle.sol:841), so the confidence term is truncated the same way.

The values below compare the code with the exact inverse and with half-up rounding at 8 decimals. They come from replaying the expression off-chain with representative prices.

| Feed (price, expo) | Code returns | Exact × 1e8 | Half-up | Relative error |
|---|---|---|---|---|
| USD/JPY 153.457 (-3) | 651648 | 651648.34 | 651648 | 5.3e-7 |
| USD/SEK 10.5483 (-4) | 9480200 | 9480200.60 | 9480201 | 6.3e-8 |
| USD/CAD 1.38127 (-5) | 72397141 | 72397141.76 | 72397142 | 1.1e-8 |
| USD/CHF 0.79641 (-5) | 125563466 | 125563466.05 | 125563466 | 4.3e-10 |

## Impact

- The error is below one 1e-8 unit per component, and always in the same direction: each inverted component is understated.
- The JPY component is affected most. At about 6.5e-3 USD per JPY it keeps only six significant digits, so its relative error reaches about 1e-6.
- The basket price is therefore slightly and systematically understated on every read. This flows into fills, marks and liquidation checks.
- The amounts are negligible, so this is reported as a precision and code-quality improvement, not a vulnerability.

## Recommendation

- Round once, at the final scale: `return (10 ** uint256(uint32(8 - expo)) + positivePrice / 2) / positivePrice;`. Alternatively, drop the dead half-up term if truncation is intended.
- Optionally, keep inverted components at 18-decimal precision until after the weighted sum. This removes the JPY granularity loss.
- Document the rounding direction next to the existing SECURITY.md rounding notes.

## Proof of Concept

## Proof of Concept
