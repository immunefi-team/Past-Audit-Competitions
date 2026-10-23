# 52837 sc insight gas heavy repeated binary search increases reward calculation gas costs

**Submitted on Aug 13th 2025 at 14:44:45 UTC by @Khay3 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52837
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>

## Description

### Summary

Repeated on-chain binary searches over dynamically growing checkpoint arrays during every reward accrual dramatically inflate gas usage, especially as the number of checkpoints grows.

### Vulnerability Detail

`PlumeRewardLogic::findRewardRateCheckpointIndexAtOrBefore` and `PlumeRewardLogic::findCommissionCheckpointIndexAtOrBefore` perform an O(log n) search **each time** `calculateRewardsWithCheckpoints()` is called. Because reward/commission checkpoints are appended on every rate change and there is **no hard cap for reward checkpoints**, the arrays can grow unbounded.

Consequently, a single user claim traverses both searches inside nested loops, compounding gas costs linearly with the number of segments processed.

Snippet from plume/src/lib/PlumeRewardLogic.sol:L629-L671:

{% code title="plume/src/lib/PlumeRewardLogic.sol:L629-L671" %}

```solidity
while (low <= high) {
    uint256 mid = low + (high - low) / 2;
    if (checkpoints[mid].timestamp <= timestamp) {
        ans = mid;
        low = mid + 1;
    } else {
        if (mid == 0) break;
        high = mid - 1;
    }
}
```

{% endcode %}

With dozens of checkpoints per validator (daily reward updates, frequent commission tweaks), claims can approach or exceed block gas limits, making reward settlement unusable.

### Impact

Large checkpoint arrays make every reward accrual and claim increasingly expensive.\
High-volume checkpoint creation can push transactions beyond block gas limits, causing denial-of-service for stakers and administrators.

### Recommendation

{% stepper %}
{% step %}

### Cache last-used checkpoint index

Cache the last-used checkpoint index per validator/token and reuse it when timestamp moves monotonically forward to avoid repeated binary searches.
{% endstep %}

{% step %}

### Limit checkpoint growth / prune

Impose a reasonable upper bound on reward checkpoint length (similar to `maxCommissionCheckpoints`) or prune old entries off-chain and migrate cumulative indices on-chain.
{% endstep %}

{% step %}

### Store cumulative indices for O(1) lookups

Optionally store cumulative indices in mappings keyed by block ranges to enable O(1) look-ups without per-call searches.
{% endstep %}
{% endstepper %}

## Proof of Concept

<details>

<summary>Proof of Concept</summary>

No Proof of Concept provided.

</details>
