# \#98249 [SC-Insight] SECURITY.md says frozen closes retain confidence validation, but the oracle replays a cached mark

**Submitted on Nov 6th 2026 at 10:14:51 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98249
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

**Type:** Documentation Improvements Insight. **Target:** Plether competition commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src/PletherOracle.sol`. This report requests a correction to `packages/perps/SECURITY.md`; it does not claim a contract exploit or loss of funds.

## Bug description

`packages/perps/SECURITY.md:655` says oracle-frozen voluntary closes “retain aggregate confidence-width validation.” At this pin, `PletherOracle._resolveCurrentBasket` returns `_frozenBasket` as soon as `policy.oracleFrozen` is true (`PletherOracle.sol:749-756`). `_frozenBasket` reads `engine.lastMarkPrice()` and `lastMarkTime()` and checks age, but does not parse or validate a supplied Pyth payload (`:729-743`). Frozen liquidation confidence is synthesized from the cached price and the configured maximum ratio (`:741-742`), irrespective of the confidence carried by the original feed observation.

The intended frozen behavior is already stated correctly in `ORACLE_SYNCHRONIZATION.md:70-80` and `WHITEPAPER.md:1071-1076`. The single sentence in the audit-facing security guide should be brought into line with those documents. The live/frozen distinction matters to a reviewer or keeper integrator deciding whether providing fresh signed Pyth data during a weekend or holiday can improve confidence or price freshness. It cannot while the oracle remains frozen.

Suggested replacement for the relevant clause:

> Aggregate basket confidence is validated and used for live/FAD execution and liquidation. During oracle-frozen operation, the oracle replays the last accepted Engine mark without parsing or validating new Pyth updates. Frozen voluntary closes use the fixed frozen-close spread instead of a Pyth adverse-confidence shift; frozen liquidations use a deterministic confidence equal to `markPrice * basketMaxConfidenceRatioBps / 10_000` for their adverse shift.

PR [#120](https://github.com/Plether-Fi/plether-core/pull/120) expressly introduced/documented cached-mark replay with no Pyth call, but left this earlier SECURITY sentence. Cecuro MED-20 and SC-M1 concern frozen divergence limits, not this text correction. Other Plether documents accurately describe the behavior, so this is a narrow documentation improvement. A private prior Insight cannot be ruled out by public search.

## Proof of Concept

## Proof of concept and conditions of value

1. At the pinned commit, read `packages/perps/SECURITY.md:655` and `packages/perps/src/PletherOracle.sol:729-756`. The documented confidence-width validation on a frozen close has no current-payload validation path: `_resolveCurrentBasket` returns before the live parser branch.
2. In a test-only Foundry fixture, install a live Engine mark of `1e8` with zero confidence, enter the Friday oracle-frozen window, commit a normal Router close, and supply a new Pyth payload at `0.90e8` with confidence `20_000_000` (about 22% of its price, far above the release 10 bps basket limit). `getOrderExecutionFee` returns zero. The normal `router.executeOrder` succeeds at the cached `1e8` mark and retains the pre-freeze publish time. This shows the supplied price and confidence were ignored; it does not assert a funds impact.
3. A separate router-pranked Oracle call in the fixture returns a frozen LONG-liquidation price of `100_025_000` from a zero-confidence cached mark. This checks the deterministic confidence rule; it is not a complete public liquidation trace.

Focused test: `packages/perps/test/perps/l16/L16DocsPromises.t.sol::test_Lead6_FrozenOracleIgnoresNewPythDataAndSynthesizesLiquidationConfidence` in the saved test-only worktree. Explicit Forge 1.5.1, `--offline --no-auto-detect --jobs 1`, **1/1 PASS**. The test source is SHA256 `39B704533377E4A64F7D0A279BBD2A712DD31EDE7E76BEB991DB2F6B05A018BE`; run log `frozen-security-doc-poc-151b.log` is SHA256 `6849EE36AD0F55F5EABE45E8C57C50CD17AEA48B71DA1DB0CB7797F397DE63B2`.

The Insight is valuable when a reader uses SECURITY.md alone to assess frozen-mode oracle guarantees. The code and other specs already disclose the correct behavior. No change to the intentional frozen execution policy is proposed.
