# \#97871 [SC-Insight] Liquidation plan decides "claim" using stale keeper carry, but live execution realizes the keeper's carry first and pays cash, so `planLiquidation` and the executed liquidation disagree

**Submitted on Nov 3rd 2026 at 17:24:45 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97871
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

### Brief/Intro

`planLiquidation` decides whether a liquidated trader's profit is paid now or recorded as a claim by looking at pool cash. The executed liquidation then runs `_checkpointBountyRecipient(keeper)`, which realizes the keeper's own carry into the pool **before** the same decision is taken again with live cash. If the keeper holds a position, the pool has more cash at execution time than the plan assumed, so the plan can say "claim" and the live path pays cash. The plan is therefore not a faithful preview of the execution it describes.

### Vulnerability Details

Plan side, cash is built only from the snapshot plus seized and collected amounts:

```solidity
// CfdEnginePlanLib.sol:1535-1543
if (delta.freshTraderPayoutAssets > 0) {
    uint256 claimsAfterPriceNettingAssets = snap.totalTraderClaimBalanceAssets - delta.pricePnlClaimConsumedAssets;
    uint256 poolCashAfterCollectionsAssets =
        snap.poolCashAssets + delta.settlementSeizedAssets + delta.actionChargeCollectedAssets;
@>  delta.freshPayoutIsImmediate = CashPriorityLib.reserveFreshPayouts(
            poolCashAfterCollectionsAssets, claimsAfterPriceNettingAssets
        ).freeCashAssets >= delta.freshTraderPayoutAssets;
@>  delta.freshPayoutCreatesClaim = !delta.freshPayoutIsImmediate;
}
```

Live side, the keeper's carry reaches the pool first, then the decision re-reads live `pool.totalAssets()`:

```solidity
// CfdEngine.sol:1336-1337  (_applyLiquidation)
if (delta.keeperBountyAssets > 0 && keeper != delta.account) {
@>  _checkpointBountyRecipient(keeper, delta.price, publishTime);   // -> _realizeCarryFromSettlement(keeper) at :215
}
// CfdEngine.sol:849 and :859-871
@> if (_canPayFreshPoolPayout(amountAssets)) { _payTraderClaim(...) } else { /* record claim */ }
```

The plan does not model the keeper's realized carry, and the live `_canPayFreshPoolPayout` does see it. The two decisions use different cash, so they can differ.

The divergence is one-directional. Plan cash is never larger than live cash at the decision point (live only adds keeper/account realized carry and the same seized and collected inflows, and claims only shrink), so the plan can say "claim" while live pays cash, but not the reverse.

### Impact Details

No funds are at risk. In my tests the liquidated trader's total value (balance + claim) was identical for a neutral keeper and a keeper with its own position, the extra cash is the keeper's real realized carry, and the pool is NAV-neutral. The effect is limited to:

- `planLiquidation` / `freshPayoutIsImmediate` (and anything built on it, such as the lens `immediatePayout` fields) is not an exact prediction of the executed result.
- The same liquidation produces a claim or an immediate payout depending only on who the keeper is.

This is why I report it as a best-practice / consistency Insight and not as a loss.

### Mitigation

Either make the plan include the keeper's pending carry (checkpoint the keeper in the planning snapshot), or run the claim-vs-cash decision once in the plan and have the live path reuse `delta.freshPayoutIsImmediate` instead of recomputing it. If the divergence is intended, document that `freshPayoutIsImmediate` is a conservative lower bound.

### References

- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1535-1543`: plan decision from snapshot cash
- `packages/perps/src/CfdEngine.sol:1336-1337`: keeper checkpoint before payout
- `packages/perps/src/CfdEngine.sol:203-216`: `_checkpointBountyRecipient` realizes keeper carry
- `packages/perps/src/CfdEngine.sol:849, 859-871`: live decision `_canPayFreshPoolPayout`
- `packages/perps/src/CfdEngineSettlementSidecar.sol:847-849`: liquidation payout goes through `settlementRecordTraderClaim`


## Proof of Concept

## Proof of Concept

Runnable code is not required for Insights, so this section lists the conditions under which the insight is valuable. I did run it on the in-scope commit at the deployed release parameters (vpiFactor 0.01e18, maxSkew 0.4, maint 10 / init 20 / frozen 300 bps, carry 500 bps) with a Foundry test; the numbers are below.

**Conditions (all must hold):**

1. A liquidated trader has a **fresh payout** (`freshTraderPayoutAssets > 0`): a position in price profit that is still liquidatable because accrued carry eats its margin.
2. Pool free cash (`pool.totalAssets() - totalTraderClaimBalanceAssets`, plus seized and collected amounts) is **just below** that fresh payout, so the plan says "claim". This needs a thin or recently drained pool (e.g. after hedged winners were paid).
3. The keeper that executes the liquidation **has its own leveraged position with unrealized carry**, large enough to close the gap when realized. Carry is charged on the financed part of a position, so the keeper's margin must be small relative to its notional.
4. The liquidation is executed with that keeper (any caller can be a keeper).

**Observed run:**

| | value |
|---|---|
| pool free cash before liquidation | 3,868.149 USDG |
| plan `freshTraderPayoutAssets` | 3,986.409 USDG |
| plan `freshPayoutIsImmediate` | `false` (claim) |
| live, neutral keeper | claim 3,986.409 USDG |
| live, keeper with own position | claim 0, paid in cash |
| trader total value (balance + claim), neutral vs keeper-with-position | 3,990.909 vs 3,990.909 USDG (identical) |

The plan's cash (free cash plus seized and collected amounts) was below the payout; the keeper's realized carry closed that gap. With a neutral keeper the same liquidation records a claim.

**When the insight matters:** integrators, keepers and UIs that rely on `planLiquidation` / the lens to decide whether a trader will be paid now or later, and any future code that assumes the plan is an exact preview of execution. It does not matter for fund safety.

