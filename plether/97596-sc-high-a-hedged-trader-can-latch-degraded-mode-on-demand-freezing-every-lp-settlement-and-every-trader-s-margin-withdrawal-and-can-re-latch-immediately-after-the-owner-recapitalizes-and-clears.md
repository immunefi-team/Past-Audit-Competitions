# \#97596 [SC-High] A hedged trader can latch degraded mode on demand, freezing every LP settlement and every trader's margin withdrawal, and can re-latch immediately after the owner recapitalizes and clears it

**Submitted on Nov 2nd 2026 at 14:08:50 UTC by @Griffinf for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97596
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description

Severity claimed: Medium. If triage accepts that the re-latch loop keeps the freeze in place for more than a day, the matching impact is "Temporary freezing of funds for at least 24 hours" (High); the PoC proves one re-latch after an owner clear, so I claim the Medium impact and state the High argument.

Degraded mode is the protocol's insolvency containment: once a close or liquidation leaves raw equity below the maximum directional liability (`E < L`), `degradedMode` latches, and only the owner can clear it after recapitalizing to `E >= L`. While latched, every trader with an open position is refused margin withdrawal, every LP epoch settlement (deposits and redemptions) reverts, and no new position can be opened. This report shows that an unprivileged trader can reach that state deliberately, cheaply, repeatedly, and with no insolvency actually present, because the admission check counts a hedged LONG and SHORT pair in full on both sides of `L = max(longMaxProfit, shortMaxProfit)` and the degraded latch fires on any profit payout that crosses `E < L`.

Precondition, stated plainly: the attacker needs capital equal to the position margins (about 1% of the pool in the PoC, returned afterwards), a basket move of at least about 0.8% in either direction (the pair is hedged, so direction is irrelevant), and pays fees and price-impact charges of 30 to 46 bps of pool size per trigger. No privileged role, no oracle manipulation, no price risk.

The expected pushback, answered: `SECURITY.md:340` and `:938-943` document that closes may latch degraded mode at `E < L` and that `L <= E < L + B` already blocks LP redemption funding. Three things are not documented and are what this report adds. First, hedged exposure is not netted: a trader holding both legs consumes all of the pool's risk capacity while carrying no net directional exposure, so the "insolvency" the latch detects is manufactured. Second, the documented recovery (owner recapitalizes to `E >= L` and calls `clearDegradedMode`) does not hold against an attacker who keeps the losing leg and part of the winning leg: the next partial close re-latches with no further price move (third PoC test). Third, degraded mode blocks LP deposits as well as redemptions and blocks every trader's free-settlement withdrawal, so the blast radius is every LP and every open trader, not only the attacker's counterparties. The program's known-issues list has no entry on degraded mode, solvency buffer sizing, or hedged exposure; the listed `SECURITY.md` sections describe the latch as a response to genuine terminal insolvency.

Asset: `packages/perps/src/CfdEngine.sol` and `packages/perps/src/libraries/SolvencyAccountingLib.sol` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, with the release parameters the program names for severity assessment (`packages/perps/DEPLOYMENT.md` at this commit: `settlementBufferBps = 25`, execution fee 0.04%, maintenance margin 0.1%, initial margin 0.2%) and the deploy script's `maxSkewRatio` 0.4 and `vpiFactor` 0.01. No configuration outside the listed deployment is needed.

## Relationship to published known issues

- Program known-issues entries "SECURITY.md: Known Limitations" and "SECURITY.md: Router and keeper limitations": neither section mentions degraded mode, the solvency buffer, hedged or same-owner exposure, or re-latching. The degraded-mode text lives in the "Critical Protocol Invariants" and "Settlement buffer" sections (`SECURITY.md:340`, `:936-943`) and describes the latch as containment for a transition that "reveals insolvency" (`:27`). A latch manufactured by a hedged pair with no net exposure is not that.
- Cecuro AI audit, SC Audit Studio pre-audit and the 2026-10-15 scan: no finding concerns hedged exposure in `L`, buffer sizing, or degraded-mode griefing. SC-M4 ("risk-free option in the 60 s oracle window") and Cecuro MED-14/16 (queued-order projection) are about admission pricing, not about `L` double-counting a hedge.
- The accepted limitation "LP redemption funding is blocked while `L <= E < L + B`" (`SECURITY.md:942`) is the state the attacker's capacity fill creates on its own. The report's claim is the step beyond it: latching, blocking deposits and trader withdrawals, and defeating the documented clear.

## Root cause

`L` is the larger of the two sides' maximum-profit envelopes, with no netting of exposure held by one owner (`libraries/SolvencyAccountingLib.sol:65-69`):

```solidity
function getMaxLiability(uint256 longMaxProfitAssets, uint256 shortMaxProfitAssets) internal pure returns (uint256) {
    return longMaxProfitAssets > shortMaxProfitAssets ? longMaxProfitAssets : shortMaxProfitAssets;
}
```

Opens require `E >= L + B` with `B = 25 bps of L`, so a hedged pair can fill capacity to `E ≈ 1.0025 L` with no net exposure. The latch fires on any close or liquidation that leaves `E < L` (`CfdEngine.sol:1351-1368`, called from `_applyClose` :1320 and `_applyLiquidation` :1344):

```solidity
uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
...
uint256 maxLiabilityAssets = _maxLiability();
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true;
    emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
}
```

After a basket move, closing part of the winning leg pays profit out of the pool, so `E` falls, while `L` is still the losing leg's full max-profit envelope. With only 25 bps of headroom, a move of about 0.8% is enough. `clearDegradedMode` (`CfdEngine.sol:658`) is owner-only and reverts `StillInsolvent` until the owner has brought `E` back to `L`; nothing stops the attacker from closing another slice afterwards.

What the latch blocks: `CfdEngineSettlementSidecar.sol:275` (`validateWithdraw` reverts `CfdEngine__DegradedMode` for any account with an open position), `HousePool.sol:1868` (`HousePool__DegradedMode` on every LP epoch settlement), `router/OrderLiquidationBatchLogic.sol:329` (`OrderRouter__DegradedMode` on every open commit).

## Vulnerability details

Paths relative to the repository root at `a82d4fef`.

**Filling capacity without exposure.** Account A opens LONG and account B opens SHORT in alternating chunks so skew never exceeds `maxSkewRatio × depth` (0.4), then a final equal pair is sized so post-open `E` sits just above `L + B`. In the PoC on a 1.002M USDG pool: `L = 999,400 USDG`, `E = 1,002,000 USDG`, free LP assets `102 USDG`. Every open passes the admission check because each is checked individually against the same `E`.

**Triggering.** After the basket moves 1% (or 0.8% with a larger close), A closes part of the winning leg through the public router. The payout lowers `E` to 998,350 USDG (1% case) while `L` stays 999,400 USDG. `_enterDegradedModeIfInsolvent` latches.

**What then fails.** An honest trader (CAROL) with a small position and spare free settlement, who could withdraw before the trigger, now gets `CfdEngine__DegradedMode`. `router.settleLpEpoch`, which succeeded for the honest junior LP's redemption request before the trigger, now reverts `HousePool__DegradedMode`. A new commit reverts `OrderRouter__DegradedMode`. The owner's `clearDegradedMode` reverts `StillInsolvent`. The test then warps one hour and one second forward and re-asserts all of it: still degraded, CAROL's withdrawal still reverts, LP settlement still reverts `HousePool__DegradedMode`, the owner still cannot clear. That is the "Temporary freezing of funds for at least 1 hour" impact, demonstrated rather than argued.

**Re-latching.** In the third test the owner recapitalizes exactly the deficit (1,050 USDG) and clears. The attacker closes 20% of the remaining winning leg with no further price move and the latch is back. The owner would need to recapitalize to `E >= L + (whatever the attacker can still pay out)`, which with the attacker holding the losing leg is the whole winning leg's unrealised profit.

**Cost.** Attacker wealth before minus after (margins, claims, balances across A and B): 3,041 USDG (30 bps of pool) at a 1% move; 4,680 USDG (46 bps) at 0.8%; 3,599 USDG (35 bps) for the re-latch sequence. Most of it is fees and price-impact charges paid to the pool. The attacker's deposits are otherwise returned.

## Impact

- **Temporary freeze of LP funds**: no LP deposit or redemption settles while latched. The honest LP's matured redemption in the PoC is stuck until the owner acts, and the attacker can undo the owner's action.
- **Temporary freeze of trader funds**: every account with an open position is refused withdrawal of free settlement that does not back the position. Traders can escape only by closing first, which forces them out of positions at the attacker's timing.
- **Market halt**: no opens. Combined with the above, the attacker can hold the protocol in containment for as long as they keep a losing leg and winning-leg slices, at tens of bps of pool per trigger and about 2.7 bps of pool per day in carry while waiting for the next move.
- No funds are stolen. The cost to the attacker is real but small relative to the frozen value (the PoC freezes a 1M pool and every trader for about 3,000 USDG).

## Recommended fix

1. Net same-owner or offsetting exposure when computing `L`, or charge the buffer per position / on total open interest rather than on `max(long, short)`, so a hedged pair cannot consume the pool's risk capacity.
2. Do not latch degraded mode on a close whose only effect on `E` is a realised profit payout with no write-off; or let the latch clear permissionlessly once `E >= L` so an owner transaction is not the only path out.
3. Raise the default `settlementBufferBps` well above 25 bps; at 25 bps a 0.8% move suffices.
4. Add an invariant test that a hedged pair cannot drive `E` below `L` through its own closes.

## Link to Proof of Concept

https://gist.github.com/aussinfosec/b38a841323422eb0e78c1134da898476

## Proof of Concept

Gist (secret): https://gist.github.com/aussinfosec/b38a841323422eb0e78c1134da898476

Three Foundry tests on a clean checkout of `a82d4fef`, on the repository's `BasePerpTest` with the Arbitrum Sepolia risk parameters. Every open, close, settlement and withdrawal goes through the public `OrderRouter`, `HousePool`, `TrancheVault` and `MarginClearinghouse` entry points.

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc && cp /path/to/gist/PoC_R1_HedgedDegradedLatch.t.sol packages/perps/test/perps/poc/
forge test --root packages/perps --match-path test/perps/poc/PoC_R1_HedgedDegradedLatch.t.sol -vv
```

Stages, all passing:

1. **Prepare.** An honest junior LP requests a 10,000 USDG redemption. Honest trader CAROL opens a small LONG and successfully withdraws 1,000 USDG of spare free settlement. Attacker accounts A and B are funded.
2. **Fill.** A and B open alternating LONG/SHORT chunks, then a final equal pair sized so `E` is just above `L + B`. Assert `E >= L + ceil(L × 25 bps)`. `router.settleLpEpoch` succeeds at this point (logged free LP assets 102 USDG).
3. **Trigger** (`test_HedgedTraderLatchesDegradedMode_1pct`, `_0_8pct`). The basket moves to 0.99 or 0.992. A closes 20% (or 76%) of its winning leg. Assert `engine.degradedMode()` is true. Then assert: CAROL's `clearinghouse.withdraw` reverts `CfdEngine__DegradedMode`; `router.settleLpEpoch` reverts `HousePool__DegradedMode`; a new commit reverts `OrderRouter__DegradedMode`; owner `clearDegradedMode` reverts `CfdEngine__StillInsolvent`. Warp `1 hours + 1`; assert `degradedMode()` is still true, CAROL's withdrawal still reverts, `router.settleLpEpoch` still reverts `HousePool__DegradedMode`, and the owner still cannot clear (logged `frozen for at least (seconds) 3602`). The attacker then exits and the latch survives; attacker cost is logged.
4. **Re-latch** (`test_HedgedTraderRelatchesAfterOwnerClear`). After the 1% trigger, the owner recapitalizes the exact deficit (1,050 USDG) and clears (assert not degraded). A closes another 20% of the remaining winning leg with no price change. Assert `engine.degradedMode()` is true again and the four reverts above hold.

Observed output:

```text
[PASS] test_HedgedTraderLatchesDegradedMode_0_8pct()
  pool assets before attack       1002000499001
  E after fill                    1002000619268
  L after fill                    999400000000
  free LP assets after fill       102119268
  E after trigger close           998963421961
  L after trigger close           999400000000
  frozen for at least (seconds)    3602
  attacker total cost (USDG atoms) 4686336415
  cost as bps of pool             46
[PASS] test_HedgedTraderLatchesDegradedMode_1pct()
  E after trigger close           998349745531
  L after trigger close           999400000000
  frozen for at least (seconds)    3602
  attacker total cost (USDG atoms) 3048092166
  cost as bps of pool             30
[PASS] test_HedgedTraderRelatchesAfterOwnerClear()
  deficit owner must recapitalize  1050254469
  frozen for at least (seconds)    3602
  attacker total cost (USDG atoms) 3605475311
  cost as bps of pool             35
```

Substitutions, stated plainly: prices come from the repository's `MockPyth` through the real `PletherOracle`; the basket move is applied as a single new tick. Carry while waiting for the move is not simulated in the cost figures (estimated at about 2.7 bps of pool per day at the deployment's 500 bps base carry).
