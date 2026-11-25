# \#98869 [SC-High] Close VPI counts the closer's own carry as pool depth (CfdEnginePlanLib.sol:638): a keeper close at zero economic depth takes the trader's whole balance for Junior LPs

**Submitted on Nov 10th 2026 at 16:57:32 UTC by @thorthumbz00 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98869
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

**Brief/Intro.** The accounting spec promises that "A zero-depth close creates no fresh VPI charge or rebate" (ACCOUNTING_SPEC.md:668-669; SECURITY.md:1439-1440). The code breaks that promise. When economic depth (pool assets minus trader claims) is zero and the closing position has any carry due, the close planner first adds the closer's own carry payment to pool assets, then reads that payment as the pool's depth. Depth becomes a few atoms, the zero-depth exemption is skipped, and the VPI curve `k * skew^2 / (2 * depth)` returns a charge far larger than the account. The planner then collects it from the trader's free balance, released margin and queued order margin. I reproduced a trader with 50,017.42 USDG ending at 0 after a keeper executed its ordinary close one second after the pool reached zero depth. In the engine-level run of the same state the pool gained 49,991.93 USDG, while the same close with no carry due leaves the trader 40,004.51 USDG.

**Details.** All lines at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`.
1. `CfdEngine.sol:1313` realizes the closer's carry before the close settles. The planner mirrors this at `libraries/CfdEnginePlanLib.sol:638`: `snap.poolAssets += realizedCarryAssets;`.
2. `libraries/CfdEnginePlanLib.sol:1339-1341` then passes `computeNetPoolDepthAssets(snap.poolAssets, snap.totalTraderClaimBalanceAssets)` as `poolDepthAssets`. At pool cash == claims, the result equals the closer's own carry, not any LP capital.
3. `libraries/CloseAccountingLib.sol:105-111` comments "With no economic depth, keep protective exits live without creating a fresh charge or rebate" but guards only `if (inputs.poolDepthAssets != 0)`. A depth of a few atoms passes.
4. `CfdMath.sol:134` computes `(skewWad * skewWad) / depthWad`. With depth in atoms and skew in thousands of USDG, the cost has no practical bound.
5. `libraries/CfdEnginePlanLib.sol:1111-1132` collects the charge from free settlement, the released liquidation reserve, released margin and, on a full close, committed order margin. Only the account's holdings cap it.

At zero depth a positive borrow base accrues carry at full utilization (ACCOUNTING_SPEC.md:665), so one second of carry is enough. Every open position closed in that state is hit. When the pool sits below claims by a gap, a position whose carry exceeds the gap is hit the same way.

**How this differs from the published scan rows S-30 and S-31 (known issue 1567).** Those Info rows say a trader who temporarily adds or removes pool liquidity can lower its own close charge (S-30) or raise its own close rebate (S-31). Both need outside liquidity manipulation, and the actor gains. This report needs no manipulation. The pool is at zero economic depth, a state the spec handles explicitly. The planner builds the depth from the victim's own carry, and the victim loses its whole balance. The violated rule is the zero-depth exemption (ACCOUNTING_SPEC.md:668), which no scan row, prior audit or known issue covers.

### Impact

Direct theft of a trader's at-rest USDG held in MarginClearinghouse. The close tx moves it to the pool, and the pool's waterfall credits it to Junior LPs. On release risk parameters:
- Keeper-executed close through OrderRouter: trader 50,017.42 -> 0 USDG in one run.
- Engine close one second after zero depth: trader 0, pool +49,991.93 USDG. The no-carry twin keeps 40,004.51 USDG.
- Pool 95.46 USDG below claims, position held 30 days: trader 50,017.42 -> 0. The seized cash then pays another trader's frozen 95.46 USDG claim in full.
- Thin depth (5,000 USDG left in the pool); this leg shows only where the seized cash goes: Junior LPs +48,948.79 of 49,999.998722 seized; the trader 100,209.39 -> 50,169.39.

The same curve also sweeps one atom above zero: at a dust depth of 130 atoms, a winning close takes 5,986.97 -> 0 USDG through the direct, keeper and armed take-profit paths, where a healthy-depth twin keeps 6,419.58 USDG. Loss per victim is bounded only by the victim's holdings. Under the undeployed-code rule I scale amounts to the minimum shown.

Demonstrated attacker gain: pool +49,991.93 USDG taken from the trader at zero depth; at thin depth (5,000 USDG left in the pool) Junior LPs +48,948.79 of 49,999.998722 seized, the trader 100,209.39 -> 50,169.39; capital committed: none beyond gas by the executing keeper (TriageEngineA.log:567-578; TriageEngineB.log:521-527)

### Risk Breakdown

Trigger: any keeper (permissionless, rules-and-trust-model.md:39-40) executes a trader's queued close or an armed take-profit. No privileged role and no price manipulation; prices are in-band Pyth prints (at most CAP_PRICE). Precondition: economic depth at zero, or a few atoms. The spec documents this operating state and keeps closes available in it (ACCOUNTING_SPEC.md:665-669). In the test, real opposite-side closes reach it. Victim action: an ordinary close request, or a take-profit armed earlier. A trader who pins `maxActionChargeAssets` is refused rather than swept; default router closes and take-profits with open close limits are swept. The loss is not a lack-of-liquidity impact: the trader is not short-paid for want of pool cash. Its own free balance, which the pool never held, is taken by a charge the spec sets to zero. A keeper acting first does not defeat this; keeper execution is the delivery path. No role can undo it: the loss settles inside `_applyClose`. `clearDegradedMode` and LP settlement pass the cash to Junior. A pause only blocks later closes. The code is in-house (`CfdMath`, `CloseAccountingLib`, `CfdEnginePlanLib`), not a vendored library.

### Recommendation

Read the close's VPI depth from economic depth before the closer's own realized carry is added at `CfdEnginePlanLib.sol:638`, so a zero-depth close stays exempt at `CloseAccountingLib.sol:107` as ACCOUNTING_SPEC.md:668 states, and bound the fresh close VPI charge so it cannot exceed what the curve charges at a minimum depth floor.

### References

- `packages/perps/src/CfdMath.sol:118-139` (VPI curve, divide by depth at :134) at a82d4fef.
- `packages/perps/src/libraries/CloseAccountingLib.sol:103-118` (zero-only guard at :107).
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:638` (own carry into pool assets), `:1328-1347` (net depth read at :1339), `:1100-1133` (collection from free, released and committed margin).
- `packages/perps/src/CfdEngine.sol:1306-1321` (`_applyClose`, carry realized at :1313).
- ACCOUNTING_SPEC.md:665-669; SECURITY.md:391, :1439-1440. In scope under rules-and-trust-model.md:39-40 (keepers untrusted).
- Prior audits and known issues: scan S-30 and S-31 cover the depth-denominator dependence with the opposite harm, at Info. Cecuro LOW-30 (stale post-VPI preview) and LOW-48 (gross vs net depth reporting) are different mechanisms. No row covers the zero-depth own-carry charge.

## Link to Proof of Concept

https://gist.github.com/tdev00/464cca71141b560a5189719b5694b796

## Proof of Concept

Foundry tests run through the repo's own `BasePerpTest` harness (and `CfdClosePreviewTestBase` for the dust legs). That harness deploys the real CfdEngine, HousePool, MarginClearinghouse and OrderRouter (`LegacyOrderRouterHarness` inherits `OrderRouter`). Only USDG (`MockSettlementToken`) and Pyth (`MockPyth`) are mocked, which the rules permit. Two identities in every leg: the victim trader, and the keeper or the LP that receives the cash. Full test files, as a secret gist (README.md, TriageEngineA.t.sol, TriageEngineB.t.sol): https://gist.github.com/tdev00/464cca71141b560a5189719b5694b796

Release configuration: the zero-depth legs (`A5ActionTest`) and the LP leg (`A4rVpiChase`) run on the release risk parameters in DEPLOYMENT.md: vpiFactor 0.01e18, maxSkewRatio 0.4e18, maintenance 10 bps, initial 20 bps, FAD 300 bps, baseCarryBps 500. The dust-depth legs (`O5StressTest`) differ, and I state it: they keep the suite harness margins (100 / 150 bps) and set baseCarryBps to 0, so the dust case needs no carry.

Steps:
1. Clone `github.com/immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` and run `forge build` once (via_ir, solc 0.8.35).
2. Download both `.t.sol` files from the gist into `poc/` at the repository root. The root `foundry.toml` remaps `@plether/perps/<File>.sol` to `packages/perps/src/<File>.sol`, and the files import `../packages/perps/test/perps/BasePerpTest.sol` and `CfdClosePreviewTestBase.sol`.
3. Run, from the repository root. The root `foundry.toml` sets `src = "integration/src"` and has no `test` key, so each command sets `FOUNDRY_TEST=poc` and skips the repo's own `test/` and `script/` trees (the form the receipts used):
   - `FOUNDRY_TEST=poc FOUNDRY_SKIP='["test/**","script/**"]' forge test --match-path poc/TriageEngineA.t.sol --match-contract A5ActionTest --match-test "test_ZeroDepth_" -vv`
   - `FOUNDRY_TEST=poc FOUNDRY_SKIP='["test/**","script/**"]' forge test --match-path poc/TriageEngineA.t.sol --match-contract O5StressTest --match-test "test_DustDepth_" -vv`
   - `FOUNDRY_TEST=poc FOUNDRY_SKIP='["test/**","script/**"]' forge test --match-path poc/TriageEngineB.t.sol --match-test test_o5F1_LpShareOfSeizedCashAfterRecovery -vv`

Lead test, `contract A5ActionTest` (release risk parameters, `TriageEngineA.t.sol:839-883`). The same book is built twice. In the twin, the drain and the victim's close happen in the same second, so no carry is due. In the run, the victim's close comes one second later:

```solidity
function test_ZeroDepth_OwnCarryBecomesDepthAndSeizesBalance() public {
    // ... twin: drain and close in one second -> owed = clearinghouse.balanceAssets(victim)
    _closeNoTick(sShort, CfdTypes.Side.SHORT, 8000, price);          // real close drains the pool
    assertEq(pool.totalAssets(), 0, "pool cash is exactly zero");
    assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no claims, so depth reads zero");
    _t();                                                              // one second: carry is due
    _closeNoTick(victim, CfdTypes.Side.LONG, 500, price);              // victim's ordinary full close
    assertGe(owed, victimFree, "with no carry due the trader keeps its free balance");
    assertEq(clearinghouse.balanceAssets(trader), 0, "every reachable atom taken");
}
```

Keeper path, `test_ZeroDepth_RouterCloseSeizesBalance` (`TriageEngineA.t.sol:1031-1048`): the victim calls `router.commitOrder(LONG, 500 lots, 0, 0, true)`, a separate `keeper` calls `router.executeOrder(closeId, pythUpdate)`, and the test asserts `status == Executed`. Claim-gap leg: `test_ZeroDepth_ClaimGapCrossedByOwnCarry` (`:886-945`). Dust legs (`O5StressTest`, `:2116-2263`): `test_DustDepth_CloseChargeTakesAllReachableCash`, `test_DustDepth_RouterClose_TraderLosesAll`, `test_DustDepth_ArmedProtectionCloseTakesAll`. LP leg (`TriageEngineB.t.sol:2110-2154`): `test_o5F1_LpShareOfSeizedCashAfterRecovery`. This leg runs at thin depth (5,000 USDG left in the pool, set by a direct token transfer at `:2122-2125`); it shows only where the seized cash goes: Junior LPs +48,948.79 of 49,999.998722 seized; the trader 100,209.39 -> 50,169.39. Real trades reach the trigger state in the A5 legs.

Captured output, verbatim (`evidence/TriageEngineA.log:553-621`, `evidence/TriageEngineB.log:521-527`; receipts `TriageEngineA.receipt.json` / `TriageEngineB.receipt.json`, exit 0, 22/22 and 51/51 passed):

```
[PASS] test_ZeroDepth_ClaimGapCrossedByOwnCarry() (gas: 11916718)
Logs:
  claims: 95460999
  pool cash: 75
  trader, carry below the gap: 49459799810
  gap: 95460924
  trader before, thirty days on: 50017423239
  trader after, thirty days on: 0
  pool cash after: 49997223314
  claims after: 95460999
  claimant paid: 95460999
  second closer before: 4995500999
  second closer after: 4664452170

[PASS] test_ZeroDepth_OwnCarryBecomesDepthAndSeizesBalance() (gas: 16428300)
Logs:
  trader, zero depth with no carry due: 40004509634
  trader, same close one second later: 0
  pool gain from the close: 49991932812

[PASS] test_ZeroDepth_RouterCloseSeizesBalance() (gas: 10111224)
Logs:
  status: 2
  terminal reason: 1
  trader before: 50017423115
  trader after: 0

Suite result: ok. 8 passed; 0 failed; 0 skipped; finished in 89.14ms (171.83ms CPU time)
...
[PASS] test_DustDepth_ArmedProtectionCloseTakesAll() (gas: 25755984)
Logs:
  balance before protection close: 5986972716
  held after protection close: 0

[PASS] test_DustDepth_CloseChargeTakesAllReachableCash() (gas: 24087365)
Logs:
  balance before close: 5986972716
  price gain on the close: 500000000
  healthy depth: 57117340496
  dust depth: 130
  zero depth: 0
  held after close, healthy twin: 6419584053
  held after close, zero depth twin: 6485172716
  held after close, dust depth: 0
  pool cash gained at dust depth: 5986972716

[PASS] test_DustDepth_RouterClose_TraderLosesAll() (gas: 24490184)
Logs:
  status: 2
  trader held before: 5986972716
  trader held after: 0
...
[PASS] test_o5F1_LpShareOfSeizedCashAfterRecovery() (gas: 5766645)
Logs:
  seized into pool 49999998722
  trader before 100209389220
  trader after 50169390498
  junior gain 48948789911
  senior gain 9137
```

(The first `...` stands for TriageEngineA.log:581-599, the A3Two suite and the O5StressTest suite header, which belong to other families. The second stands for the rest of the TriageEngineA log and TriageEngineB.log:1-520, also other families.)

### Before / after state delta (USDG, 6 decimals)

| Account / state | Before | After | Delta |
|---|---|---|---|
| Trader, keeper-executed close at zero depth (same run) | 50,017.42 | 0 | -50,017.42 |
| Trader, close one second after zero depth (vs no-carry twin) | 40,004.51 (twin) | 0 | -40,004.51 |
| Pool, same close | - | - | +49,991.93 |
| Trader, 30-day hold, pool 95.46 below claims (same run) | 50,017.42 | 0 | -50,017.42 |
| Trader, dust depth 130 atoms, direct / keeper / take-profit | 5,986.97 | 0 | -5,986.97 (healthy twin keeps 6,419.58) |
| Thin depth (5,000 USDG left in the pool), where the seized cash goes | trader 100,209.39 | trader 50,169.39 | Junior LPs +48,948.79 of 49,999.998722 seized |
