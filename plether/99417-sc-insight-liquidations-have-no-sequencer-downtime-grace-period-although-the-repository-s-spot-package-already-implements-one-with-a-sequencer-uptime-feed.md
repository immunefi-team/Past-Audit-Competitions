# \#99417 [SC-Insight] Liquidations have no sequencer-downtime grace period, although the repository's spot package already implements one with a sequencer uptime feed

**Submitted on Nov 13th 2026 at 15:27:40 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99417
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Description

**Category: Insight - Security best practices.** No vulnerability, loss or exploit is claimed; this is a recommendation to adopt a pattern already used elsewhere in the same repository.

The perps package contains no reference to a sequencer uptime check: a search of `packages/perps/src` for "sequencer" finds nothing. Liquidation pricing is resolved in `PletherOracle._updateLiquidationSnapshot` (`PletherOracle.sol` lines 532-543), which applies the liquidation freshness policy (`liquidationStalenessLimit`, 15 seconds by constructor default, line 81) to a signed Pyth basket or the frozen mark, and `OrderLiquidationBatchLogic.sol` and `OrderRouter` execute the liquidation. Nothing in that path knows whether the Arbitrum sequencer has just come back from downtime.

The spot package in the same repository does. `packages/spot/src/core/SyntheticSplitter.sol` declares `AggregatorV3Interface public immutable SEQUENCER_UPTIME_FEED` and `SEQUENCER_GRACE_PERIOD = 1 hours` (lines 77-79), passes both to `OracleLib.getValidatedPrice(ORACLE, SEQUENCER_UPTIME_FEED, SEQUENCER_GRACE_PERIOD, ORACLE_TIMEOUT)` (line 848) and sets the feed in its constructor (line 164). So the project already treats "the sequencer was down, give users a window to react" as a price-validation requirement for the spot contracts.

`SECURITY.md` section "Arbitrum and Pyth clock alignment" (lines 701-714) addresses the sequencer timestamp trailing Pyth, not sequencer outages or restarts.

Two properties of the existing design mean this gap is not already covered by another control. First, Pyth's own 15-second `liquidationStalenessLimit` does not substitute for a sequencer-uptime check: Pyth's off-chain network (Hermes) publishes continuously regardless of Arbitrum's sequencer state, so the moment the sequencer resumes after any outage, anyone can permissionlessly push an on-chain update carrying a Pyth timestamp under 15 seconds old, satisfying the staleness check immediately — independent of how long the sequencer itself was down or how long affected users had no ability to submit a transaction. Second, both liquidation entry points are themselves permissionless: `OrderRouter.executeLiquidation` and `executeLiquidationBatch` are `external payable nonReentrant` with no access-control modifier, and the latter's own NatSpec states "Permissionlessly attempts up to 256 account liquidations" (`OrderRouter.sol` lines 493, 513). An operational convention (e.g. the project's own keepers waiting out a grace window before submitting) would not close the gap on its own, since any other party can call these functions directly the moment the sequencer resumes.


## Proof of Concept

## Proof of Concept

**Conditions under which this Insight is valuable.**

1. Sequencer downtime is a recognized L2 event: while the sequencer is down, users cannot submit transactions (adding margin, closing, cancelling) but a fresh signed Pyth payload is available to whoever submits a liquidation first after restart. The liquidation freshness check (15 s) is satisfied by a payload obtained at restart, so liquidations are enabled immediately while the users whose positions moved during the outage had no chance to respond. A grace window after restart removes that asymmetry; it is the standard reason the spot contracts use one.
2. The recommendation is most valuable for the Friday-to-Sunday reopening and any other time a gap in prices coincides with an outage; it is worth less if governance considers instant liquidation after downtime the safer policy (bad-debt risk in a fast market), which is a genuine trade-off between user fairness and solvency that the maintainers should decide and document either way.
3. The implementation cost is low because the pattern, the feed interface and the helper already exist in the repository, and a grace check would apply to liquidation execution only, leaving closes and risk-reducing actions untouched.

Read-only reproduction at the frozen commit: search `packages/perps/src` for "sequencer" (no hits); read `PletherOracle.sol` lines 532-543 and `packages/spot/src/core/SyntheticSplitter.sol` lines 77-79, 164 and 846-849.

