# \#99916 [SC-Insight] Missing per-leg price bounds in PletherOracle allows extreme Pyth quotes (price = 1) to silently distort the basket mark

**Submitted on Nov 15th 2026 at 18:04:03 UTC by @SuitableAstrix for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99916
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Brief/Intro
`PletherOracle.sol` enforces no per-leg minimum or maximum price sanity bounds relative to `basePrices[i]` when aggregating its 6-asset Pyth FX basket. If an upstream Pyth publisher/aggregator malfunction emits the smallest valid positive quote (`price = 1` at `expo = -8`) alongside a tight confidence interval, `PletherOracle` silently truncates a non-inverted leg's contribution to zero (dropping the basket mark by -57.6%, from 1.00 to 0.424) or pins an inverted leg to `CAP_PRICE` (+100% jump to 2.00) instead of reverting. Because the oracle does not fail closed, downstream execution and liquidation logic (`OrderRouter` and `CfdEngine`) treat the distorted mark as valid price discovery rather than holding orders in a safe `Pending` state.

## Vulnerability Details
`PletherOracle` computes the 6-asset FX basket mark as a weighted sum of normalized component prices, where par equals 100,000,000 (1.00 in 8-decimal precision):

```solidity
weighted = (norm * quantities[i]) / (basePrices[i] * CHAINLINK_TO_TOKEN_SCALE);
```

During feed validation across both live (`updatePrice`) and historical (`updateOrderExecutionPrice`) entrypoints, `PletherOracle` enforces four checks:

1. **Strictly positive quotes:** Any leg with `price <= 0` reverts with `PletherOracle__InvalidPrice(feedId, price)`.
2. **Bounded inverse confidence:** Any inverted leg (`inversions[i] == true`) with `conf >= price` reverts with `PletherOracle__InvalidInverseConfidence(feedId, price, conf)`.
3. **Exponent scaling bounds:** Extreme exponents that overflow `10**(26 - expo)` or `10**(-8 - expo)` fail closed with an EVM arithmetic overflow (`Panic(0x11)`).
4. **Aggregate ceiling clamp:** If the final basket price exceeds `CAP_PRICE` (200,000,000, or 2.00), the oracle clamps the returned mark to `CAP_PRICE` rather than reverting.

However, `PletherOracle` enforces no per-leg minimum or maximum price bounds around `basePrices[i]`. The smallest valid positive Pyth price is `price = 1` (`1e-8` at `expo = -8`), which passes `price > 0` and triggers two silent distortions depending on whether the feed is inverted:

- **Non-Inverted Leg Silent Truncation (`inversions[i] == false`):** When `price = 1` at `expo = -8`, `norm` evaluates to 1. Because `quantities[i]` is smaller than `basePrices[i] * CHAINLINK_TO_TOKEN_SCALE`, integer division truncates `(1 * quantities[i]) / (basePrices[i] * CHAINLINK_TO_TOKEN_SCALE)` to 0. On Leg 0 (5,760 bps / 57.60% basket weight), its normal par contribution of `57,600,000` (`0.576`) vanishes completely, dropping the accepted basket mark from 1.00 to 0.424 (-57.6%) in a single block with zero revert.
- **Inverted Leg `CAP_PRICE` Pinning (`inversions[i] == true`):** For an inverted leg, normalization computes `scaledInverse = ((10**(26 - expo)) + raw / 2) / raw` followed by `norm = scaledInverse / 1e18`. At `expo = -8` and `raw = 1`, `scaledPrecision = 1e34`, producing `norm = 1e16`. Multiplying `1e16` by `quantities[i]` drives the raw basket sum far above `CAP_PRICE` (200,000,000), causing the oracle to silently clamp the accepted mark to 2.00 (+100% above par) instead of rejecting the anomalous quote.

### Recommended Solution
1. **Enforce Per-Leg Deviation Bounds Around `basePrices[i]`:** Add configurable ratio bounds (`minLegPriceRatioBps` and `maxLegPriceRatioBps`, e.g., 5,000 bps / 0.5x to 20,000 bps / 2.0x of `basePrices[i]`) in `_computeBasketPriceFromFeeds` so single-feed near-zero or spike quotes revert explicitly:

```solidity
uint256 minAllowed = (basePrices[i] * minLegPriceRatioBps) / 10_000;
uint256 maxAllowed = (basePrices[i] * maxLegPriceRatioBps) / 10_000;
if (norm < minAllowed || norm > maxAllowed) {
    revert PletherOracle__LegPriceOutOfBounds(feedId, norm, minAllowed, maxAllowed);
}
```

2. **Fail Closed on `CAP_PRICE` Breach:** Consider reverting with `PletherOracle__BasketPriceOutOfBounds` when the un-clamped basket sum exceeds `CAP_PRICE` (or falls below a symmetric `FLOOR_PRICE`) rather than silently clamping to 2.00, as hitting a 2x ceiling on a major G6 FX basket signals broken feed data rather than a tradable market quote.

## Impact Details
This finding qualifies under **Security Best practices** (`Insight`) as a defense-in-depth hardening improvement against upstream oracle anomalies:

Because Pyth updates are cryptographically signed by Pyth's validator network (`parsePriceFeedUpdates` / `parsePriceFeedUpdatesUnique`), an unprivileged user cannot forge `price = 1`. Exploitation therefore requires an upstream Pyth publisher/aggregator malfunction where a feed publishes `price = 1` alongside a tight confidence interval (`conf < price` on inverted legs and aggregate confidence <= 10 bps). If that upstream fault occurs:

- **Instantaneous Mass Liquidations:** A -57.6% drop (1.00 -> 0.424) immediately pushes leveraged `SHORT` positions into liquidation, while a +100% jump (1.00 -> 2.00) immediately liquidates leveraged `LONG` positions.
- **Bad Debt & LP Principal Erosion:** Positions gapped far past their `pnlPledgeAssets` generate uncollectible write-offs (`PriceLossWrittenOff`), while open positions on the winning side of the distorted mark can close profitably before Pyth recovers, creating trader claims against `HousePool` liquidity.
- **Silent Acceptance Instead of Fail-Closed Pending State:** Because clamping to `CAP_PRICE` or truncating to 0 does not revert, `OrderRouter` and `CfdEngine` treat the distorted mark as valid price discovery instead of holding orders in a safe `Pending` state.

## References
- **In-Scope Asset:** [packages/perps/src](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src)
- **Target Contract:** `packages/perps/src/PletherOracle.sol` (`0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2` on Arbitrum Sepolia, chainId: 421614, fork-verified at block 328,625,574)
- **Affected Functions:** `updatePrice`, `updateOrderExecutionPrice`, `_computeBasketPriceFromFeeds`

## Link to Proof of Concept

https://gist.github.com/Duarte-Master/4e15e01fd67a2ec9163d5341765f8d12

## Proof of Concept


## Proof of Concept

### 1. Conditions Under Which This Insight Is Valuable
Adding per-leg price deviation bounds (`minLegPriceRatioBps` and `maxLegPriceRatioBps`) in `PletherOracle._computeBasketPriceFromFeeds` provides critical defense-in-depth whenever all of the following conditions hold simultaneously:

1. **Upstream Pyth Feed Anomaly (price = 1):** A Pyth publisher or aggregator malfunction emits the smallest valid positive quote (`price = 1` at `expo = -8`, representing `1e-8`) on one of the six FX basket feeds, signed validly by Pyth's validator network (`parsePriceFeedUpdates` / `parsePriceFeedUpdatesUnique`).
2. **Tight Reported Confidence Interval:** The anomalous leg satisfies `conf < price` (if inverted) and the weighted basket confidence remains within `basketMaxConfidenceRatioBps` (<= 10 bps), allowing the quote to pass `PletherOracle`'s existing confidence checks.
3. **Absence of Per-Leg Bounds Around `basePrices[i]`:** Because `PletherOracle` only checks `price > 0`, exponent overflow bounds, and the final aggregate `CAP_PRICE` (200,000,000), neither an extreme downward quote (1e-8) nor an extreme upward inverted quote triggers a revert.

Under normal market conditions, major G6 FX pairs never move by > 50% or > 200% within a single oracle update window. Enforcing configurable per-leg ratio bounds around `basePrices[i]` therefore carries zero operational cost during normal price discovery while guaranteeing fail-closed behavior during upstream feed faults.

---

### 2. Step-by-Step Execution & Arithmetic Trace

**Target Deployment & Fork State:**
- **Network:** Arbitrum Sepolia (chainId: 421614)
- **Pinned Block:** 328,625,574
- **Contract:** `PletherOracle` (`0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2`)

#### Scenario A: Non-Inverted Leg Silent Truncation (-57.6% Basket Drop)
1. **Par Baseline:** Under normal quotes across all 6 FX legs, the weighted sum in `_computeBasketPriceFromFeeds` equals 100,000,000 (1.00 in 8-decimal precision), with Leg 0 contributing 57,600,000 (5,760 bps / 57.60% weight).
2. **Anomalous Input:** Leg 0 (`inversions[0] == false`) receives a signed Pyth update with `price = 1`, `expo = -8`, and `conf = 0`, while Legs 1–5 remain at par.
3. **Validation Bypass:** `price > 0` succeeds (`1 > 0`), exponent scaling succeeds without overflow, and `norm` evaluates to 1.
4. **Integer Division Truncation:** `PletherOracle` computes Leg 0's weighted contribution as:
   ```solidity
   weighted = (1 * quantities[0]) / (basePrices[0] * CHAINLINK_TO_TOKEN_SCALE);
   ```
   Because `quantities[0] < basePrices[0] * CHAINLINK_TO_TOKEN_SCALE`, integer division truncates `weighted` to 0.
5. **Resulting State:** Leg 0's 57,600,000 contribution vanishes without reverting. `updatePrice` / `updateOrderExecutionPrice` accepts a composite basket mark of 42,400,000 (0.424), an instantaneous -57.6% drop from 1.00.

#### Scenario B: Inverted Leg `CAP_PRICE` Pinning (`+100%` Basket Jump)
1. **Anomalous Input:** An inverted leg (`inversions[i] == true`) receives a signed Pyth update with `price = 1`, `expo = -8`, and `conf = 0`, while the remaining five legs remain at par.
2. **Validation Bypass:** `price > 0` (`1 > 0`) and `conf < price` (`0 < 1`) both pass.
3. **Inverse Scaling Spike:** `_computeBasketPriceFromFeeds` computes:
   ```solidity
   scaledInverse = ((10**(26 - (-8))) + 1 / 2) / 1; // 1e34
   norm = scaledInverse / 1e18;                      // 1e16
   ```
   Multiplying `norm = 1e16` by `quantities[i]` produces a raw basket sum far exceeding `CAP_PRICE` (`200,000,000`).
4. **Silent Clamping:** Instead of reverting on the anomalous input, `PletherOracle` clamps the returned mark to `CAP_PRICE` (200,000,000, or 2.00), an instantaneous +100% jump above par.

---

### 3. System-Wide Impact of the Proposed Fix
- **Without Per-Leg Bounds (Current Behavior):** `PletherOracle` accepts 0.424 or 2.00 as valid price discovery, causing `OrderRouter` and `CfdEngine` to execute queued orders and trigger liquidations at a distorted mark rather than holding orders in a safe `Pending` state.
- **With Per-Leg Deviation Bounds (Recommended):** `_computeBasketPriceFromFeeds` reverts immediately with `PletherOracle__LegPriceOutOfBounds(feedId, norm, minAllowed, maxAllowed)` whenever `norm` falls outside `[minAllowed, maxAllowed]` relative to `basePrices[i]`, isolating downstream clearinghouse margin and `HousePool` liquidity until valid Pyth quotes resume.
