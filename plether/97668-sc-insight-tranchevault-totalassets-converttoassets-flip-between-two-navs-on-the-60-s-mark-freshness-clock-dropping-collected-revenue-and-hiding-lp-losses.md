# \#97668 [SC-Insight] TrancheVault totalAssets / convertToAssets flip between two NAVs on the 60 s mark-freshness clock, dropping collected revenue and hiding LP losses

**Submitted on Nov 2nd 2026 at 15:57:10 UTC by @janneh2000 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97668
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

I found that the tranche vaults' ERC-4626 views report two different NAVs depending only on whether the engine mark is younger than HousePool's markStalenessLimit (60 s by default, HousePool.sol:252). Nothing economic happens in between. This is in audit-comp-plether at a82d4fef. Settlement pricing is not affected, because it always runs on a fresh mark. But anything that values tranche shares from the views reads a share price that depends on how recently a keeper touched the oracle: wallets, dashboards, share-price oracles, lending or collateral integrations.

TrancheVault.totalAssets (TrancheVault.sol:510-513) and convertToAssets (:521-525) read HousePool.getPendingTrancheState (HousePool.sol:1295-1323). With any open position, mark freshness is required (CfdEngineProtocolLens.sol:74-81, HousePoolFreshnessLib.sol:19-31). When the mark is stale, the reconcile plan stops right after the senior coupon:

```solidity
// packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:64-66 @ a82d4fef
if (!markFresh) {
    return plan;
}
```

So the views return the last settled principal plus coupon. They leave out collected cash revenue (VPI, carry, fees), realised payouts and mark-to-market. estimateRedeemAssets (TrancheVault.sol:1385-1390) uses the same value.

Measured with the PoC below (10M pool, one honest 3.6M LONG open, deployment risk params):
- Revenue: right after a mark write, junior + senior totalAssets equal the pool's USDG balance, 10,008,478.70. 61 s later, with no call in between, they read 10,002,000.00, dropping 6,478.70 USDG of VPI already collected in cash. A same-price updateMarkPrice brings it back, and 61 s later it drops out again.
- Losses: after a -0.5% move in the trader's favour, the fresh views show LPs at 9,990,478.70. 61 s later the stale views read 10,002,000.00, which overstates LP value by 11,521.30 USDG. One LP's convertToAssets reads 7,999,981.37 instead of 7,988,461.83.

When this matters (conditions):
- At least one open position, which is normal on a live market.
- More than 60 s since the last mark write. On weekdays that is true in every block without an execution, liquidation, updateMarkPrice or LP settlement, so on a quiet deployment it holds most of the time.
- A reader that prices tranche shares from totalAssets or convertToAssets. The overstatement after LP losses is the dangerous case for a lending or collateral integration. The understatement after revenue hurts anyone marking or selling shares.

Why this isn't already known: the getPendingTrancheState NatSpec (HousePool.sol:1285-1290) says "mark-dependent waterfall changes apply only when the applicable mark is fresh". SECURITY.md's LP accounting limitations say "exact terminal NAV is only as current as its authenticated mark". Neither says the views drop mark-independent cash revenue, swing on a 60 s clock, or overstate after LP losses. "As current as its mark" reads as valuing at the cached mark, which is not what the code does. No audit text or scan finding covers the tranche share-price views. Cecuro LOW-16 is about withdrawal-readiness flags, not share price.

Recommendation:
- (a) When the mark is stale, value the pending state at the cached mark, as frozen-mode exits already do, so the views stay continuous.
- (b) Or expose a NAV freshness flag and timestamp on the vaults, and document that integrators must check it.
- (c) At minimum, recognise mark-independent collected cash (VPI, carry, fees) in the preview, so the stale view never drops revenue already in the pool.

PoC (runnable, although Insights don't require it): one Foundry file next to the repo's BasePerpTest, using the real engine, HousePool, TrancheVaults and OrderRouter, with MockPyth only and src unmodified. [PoC link]

```
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
cp /path/to/TrancheViewStaleness_Insight.t.sol packages/perps/test/perps/
forge test --root packages/perps --match-path test/perps/TrancheViewStaleness_Insight.t.sol -vv
```

```
[PASS] test_1_CollectedRevenueFlipsOnTheFreshnessClock()
    pool USDG balance                          10008478.70 USDG
    junior+senior totalAssets, mark fresh      10008478.70 USDG
    junior+senior totalAssets, 61 s later      10002000.00 USDG
    after updateMarkPrice at the same price    10008478.70 USDG
    61 s later again                           10002000.00 USDG
    view gap (fresh - stale)                   6478.70 USDG
[PASS] test_2_LpLossHiddenWhenMarkStale()
    junior+senior totalAssets, fresh at 0.995  9990478.70 USDG
    junior+senior totalAssets, 61 s later      10002000.00 USDG
    LP convertToAssets, fresh at 0.995         7988461.83 USDG
    LP convertToAssets, 61 s later             7999981.37 USDG
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

Local testing only; no transaction was sent to any network.



## Link to Proof of Concept

https://gist.github.com/janneh2000/1dce6e15caa86b67107c2f4c0284ac37

## Proof of Concept

## Proof of Concept

PoC (runnable, although Insights don't require it): one Foundry file next to the repo's BasePerpTest, using the real engine, HousePool, TrancheVaults and OrderRouter, with MockPyth only and src unmodified. [PoC link]

```
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
cp /path/to/TrancheViewStaleness_Insight.t.sol packages/perps/test/perps/
forge test --root packages/perps --match-path test/perps/TrancheViewStaleness_Insight.t.sol -vv
```

```
[PASS] test_1_CollectedRevenueFlipsOnTheFreshnessClock()
    pool USDG balance                          10008478.70 USDG
    junior+senior totalAssets, mark fresh      10008478.70 USDG
    junior+senior totalAssets, 61 s later      10002000.00 USDG
    after updateMarkPrice at the same price    10008478.70 USDG
    61 s later again                           10002000.00 USDG
    view gap (fresh - stale)                   6478.70 USDG
[PASS] test_2_LpLossHiddenWhenMarkStale()
    junior+senior totalAssets, fresh at 0.995  9990478.70 USDG
    junior+senior totalAssets, 61 s later      10002000.00 USDG
    LP convertToAssets, fresh at 0.995         7988461.83 USDG
    LP convertToAssets, 61 s later             7999981.37 USDG
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

**LOCAL CLI**: 

rivaldo@Alies-MacBook-Air INSIGHT-tranche-view-staleness % cd ~/Downloads/VPI-LP-NAV/audit-comp-plether/audit-comp-plether
cp ~/Downloads/INSIGHT-tranche-view-staleness/submission/TrancheViewStaleness_Insight.t.sol packages/perps/test/perps/
forge test --root packages/perps --match-path test/perps/TrancheViewStaleness_Insight.t.sol -vv
[⠒] Compiling...
[⠒] Compiling 1 files with Solc 0.8.35
[⠆] Solc 0.8.35 finished in 49.28s
Compiler run successful with warnings:


Ran 2 tests for test/perps/TrancheViewStaleness_Insight.t.sol:TrancheViewStalenessInsightTest
[PASS] test_1_CollectedRevenueFlipsOnTheFreshnessClock() (gas: 5217093)
Logs:
  REVENUE DIRECTION (open 3.6M LONG paid VPI + fee; price 1.0 throughout):
    pool USDG balance                          10008478.70 USDG
    junior+senior totalAssets, mark fresh      10008478.70 USDG
    junior+senior totalAssets, 61 s later      10002000.00 USDG
    after updateMarkPrice at the same price    10008478.70 USDG
    61 s later again                           10002000.00 USDG
    view gap (fresh - stale)                   6478.70 USDG
    LP convertToAssets, fresh                  8006459.59 USDG
    LP convertToAssets, stale                  7999981.38 USDG

[PASS] test_2_LpLossHiddenWhenMarkStale() (gas: 5224851)
Logs:
  LOSS DIRECTION (same open LONG; all six feeds move -0.5%):
    junior+senior totalAssets, fresh at 1.000  10008478.70 USDG
    junior+senior totalAssets, fresh at 0.995  9990478.70 USDG
    junior+senior totalAssets, 61 s later      10002000.00 USDG
    LP convertToAssets, fresh at 0.995         7988461.83 USDG
    LP convertToAssets, 61 s later             7999981.37 USDG
    => LPs lost on the move, and the stale view overstates them

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 55.01ms (58.26ms CPU time)

Ran 1 test suite in 62.92ms (55.01ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)

