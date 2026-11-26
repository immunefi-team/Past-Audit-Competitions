# \#98439 [SC-Insight] PriceSnapshot NatSpec labels the newest component timestamp as the earliest

**Submitted on Nov 8th 2026 at 03:34:13 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98439
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Classification

This is one **Documentation Improvements Insight** about the meaning of the public oracle snapshot timestamp. It is not a Critical, High, Medium, or Low vulnerability report, and it does not claim current loss, exploitable protocol behavior, or funds at risk.

The discrepancy is semantic rather than grammatical: the public interface says a returned field is the minimum component time, while the implementation returns the maximum component time. A keeper, monitor, or policy integration can therefore implement a freshness or lower-bound check incorrectly while following the NatSpec exactly.

## Public NatSpec contradicts the implementation

At frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, [`IPletherOracle.sol`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPletherOracle.sol) documents both public snapshot structures as follows:

```solidity
/// @param publishTime Earliest component publish time as a Unix timestamp.
struct PriceSnapshot {
    ...
    uint64 publishTime;
    ...
}

/// @param publishTime Earliest component publish time as a Unix timestamp.
struct LiquidationBatchSnapshot {
    ...
    uint64 publishTime;
    ...
}
```

[`PletherOracle.sol`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol) instead tracks both extrema and assigns them separately:

```solidity
basket.publishTime = uint64(maxPublishTime);
basket.oldestPublishTime = uint64(minPublishTime);
```

`_snapshotFromBasket` then returns the maximum through the documented public field:

```solidity
snapshot.publishTime = basket.publishTime;
```

The implementation's own internal `BasketPrice` NatSpec calls these values `Newest component publish time` and `Oldest component publish time`. [`ORACLE_SYNCHRONIZATION.md`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md) likewise says that an accepted basket's publish time is the newest component time, and `SECURITY.md` says the newest component time advances the protection clock. The two public struct comments are the stale definitions.

## Why the correction has concrete security value

Suppose an accepted multi-feed basket contains component times `T-d` and `T`, where `d` is within the applicable divergence policy. An integration that follows the public NatSpec treats `publishTime` as `T-d`. The implementation actually returns `T`.

If that integration checks `publishTime >= T-k` for some `0 < k < d`, the documented minimum-component check appears to pass even though the actual oldest component is below the chosen boundary. This affects how an external integration can reason about freshness, epoch floors, and temporal ordering; it is not a cosmetic rephrasing.

The repository itself demonstrates that this comparison shape is realistic: `SettlementMonitorLensSidecar` decodes the public field and compares it with a minimum publish time. That current PoolReconcile path is safe because `PletherOracle` independently rejects the basket unless `minPublishTime` is at least the current epoch start. I am not claiming that monitor, or any other current Plether consumer, is exploitable. The example shows why a public `min`-versus-`max` definition must be exact for other integrations that do not have that independent check.

The accepted difference between the oldest and newest component is bounded by the selected mode and configuration. The repository documents a default five-second historical component-divergence limit and a 15-second live parsing lookback. Those are context for the possible semantic gap, not a claim that every snapshot has either discrepancy.

## Recommendation

Change both public struct comments to:

```solidity
/// @param publishTime Newest component publish time as a Unix timestamp.
```

Where an integration needs the oldest component time, expose or return the separately tracked `oldestPublishTime`, or provide a getter whose NatSpec clearly distinguishes the two extrema. Add a documentation test or source assertion that the public return-field description stays aligned with `_snapshotFromBasket` and the basket aggregation assignment.

## Public overlap and limitations

The retained public audit discusses component-time skew in an older implementation, which then assigned the minimum time to `basket.publishTime`. It does not identify the current stale public struct definitions after the accepted-basket rewrite changed that assignment to the maximum. Commit `819192182833597641504b38cce7751bbcacecb8` introduced the current maximum-time assignment.

No exact prior report for this public NatSpec mismatch was found in the retained submission corpus. Current public and private duplicate state must still be checked by the platform at filing. This report does not predict acceptance or reward.


## Proof of Concept

## Proof of Concept

## Conditions under which this Insight is valuable

This documentation correction is valuable when all of the following are true:

1. A keeper, monitor, UI, policy engine, or downstream contract consumes `PriceSnapshot.publishTime` or `LiquidationBatchSnapshot.publishTime`.
2. The consumer follows the public NatSpec and interprets the field as the earliest component timestamp.
3. The accepted basket contains components with different timestamps.
4. The consumer uses the field in a minimum-time, freshness, epoch-floor, or ordering decision without separately reading or validating every component timestamp.

No runnable PoC is necessary to demonstrate this documentation mismatch. The source-level demonstration below uses the exact frozen assignments.

## Text demonstration

Let two component observations have timestamps:

```text
oldest component = T - d
newest component = T
```

where `d > 0` is within the mode's accepted component-divergence limit.

The interface promises:

```text
snapshot.publishTime = earliest component = T - d
```

The implementation computes:

```text
minPublishTime = T - d
maxPublishTime = T
basket.publishTime = maxPublishTime = T
basket.oldestPublishTime = minPublishTime = T - d
snapshot.publishTime = basket.publishTime = T
```

Now consider a consumer that follows the interface and requires every component to be at least `T-k`, with `0 < k < d`. It implements the documented conservative check:

```text
snapshot.publishTime >= T - k
```

On the returned value, the check becomes:

```text
T >= T - k                 => true
```

But the condition the consumer believed it checked is:

```text
T - d >= T - k             => false because d > k
```

The implementation is internally consistent with newest-time semantics; the error is that the public NatSpec tells the consumer to assign oldest-time meaning to that value.

## Source reconciliation

The mismatch can be reproduced by reading these frozen locations:

- `packages/perps/src/interfaces/IPletherOracle.sol`: the two public `publishTime` comments say `Earliest`.
- `packages/perps/src/PletherOracle.sol`: `_computeBasketPriceFromFeeds` stores the maximum in `basket.publishTime` and the minimum in `basket.oldestPublishTime`.
- `packages/perps/src/PletherOracle.sol`: `_snapshotFromBasket` copies `basket.publishTime` to `snapshot.publishTime`.
- `packages/perps/ORACLE_SYNCHRONIZATION.md`: the accepted basket documentation says its publish time is the newest component time.

As a matched safety check, the current PoolReconcile implementation explicitly tests `minPublishTime` against the epoch boundary before returning a snapshot. That independent minimum-component check means this report is not alleging that Plether's own epoch settlement currently accepts a pre-boundary component. The Insight is the incorrect public contract presented to integrations that may not reproduce that hidden internal check.

## Expected documentation result

After the correction, an integrator can distinguish:

```text
publishTime       = newest / maximum component time
oldestPublishTime = oldest / minimum component time, where separately exposed
```

That definition matches the aggregation code, internal NatSpec, synchronization guide, and runtime consumers.



