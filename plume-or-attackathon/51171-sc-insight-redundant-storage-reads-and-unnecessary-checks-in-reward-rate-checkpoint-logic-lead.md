# 51171 sc insight redundant storage reads and unnecessary checks in reward rate checkpoint logic lead to inefficient gas usage

**Submitted on Jul 31st 2025 at 18:10:39 UTC by @farman1094 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51171
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:** Inefficient gas usage due to redundant storage reads and unnecessary checks

## Description

### Brief / Intro

There is an optimization opportunity in the `PlumeRewardLogic` contract: the validator reward rate checkpoint array is read from storage multiple times within the same execution path. Additionally, a zero-length check is performed in a child function while the parent function already guarantees existence. These issues do not affect correctness, but they cause unnecessary SLOADs which increase gas costs.

### Vulnerability Details

In `getEffectiveRewardRateAt` and `findRewardRateCheckpointIndexAtOrBefore` functions, the code repeatedly retrieves the checkpoints from storage:

```solidity
PlumeStakingStorage.RateCheckpoint[] storage checkpoints = $.validatorRewardRateCheckpoints[validatorId][token];
```

These arrays are only read (no writes). Reading them once into memory in the parent and passing the memory array to the child would avoid repeated SLOADs:

```solidity
PlumeStakingStorage.RateCheckpoint[] memory checkpoints = $.validatorRewardRateCheckpoints[validatorId][token];
uint256 idx = findRewardRateCheckpointIndexAtOrBefore(checkpoints, timestamp);
```

Also, the child currently contains a redundant zero-length check:

```solidity
if (len == 0) {
    return 0; // Indicates no checkpoints, caller might use global rate.
}
```

If the parent already verifies that checkpoints exist, this check is unnecessary and can be removed to improve clarity and efficiency.

### Impact Details

No direct exploit leads to fund theft or privilege escalation. The issue increases gas consumption for callers interacting with staking-related functions, reducing efficiency and potentially increasing user costs.

## Proof of Concept

{% stepper %}
{% step %}

### PoC — call path leading to redundant reads

A user (or contract) calls a function that ends up calling:

```solidity
getEffectiveRewardRateAt($, validatorId, token, timestamp)
```

Inside `getEffectiveRewardRateAt`, the checkpoints are read from storage:

```solidity
PlumeStakingStorage.RateCheckpoint[] storage checkpoints = $.validatorRewardRateCheckpoints[validatorId][token];
```

This SLOAD costs gas proportional to the array size.
{% endstep %}

{% step %}

### PoC — parent calls child without passing memory array

The parent calls the helper:

```solidity
uint256 idx = findRewardRateCheckpointIndexAtOrBefore($, validatorId, token, timestamp);
```

The child does not receive the previously loaded array, so it fetches the same storage again.
{% endstep %}

{% step %}

### PoC — child repeats the storage read

Inside `findRewardRateCheckpointIndexAtOrBefore` the same storage fetch occurs:

```solidity
PlumeStakingStorage.RateCheckpoint[] storage checkpoints = $.validatorRewardRateCheckpoints[validatorId][token];
```

This is a second read of the same storage data, incurring extra gas.
{% endstep %}

{% step %}

### PoC — read-only operations after redundant reads

Both functions then perform a binary search and timestamp comparisons on the array. Since no mutation occurs, the repeated storage reads are unnecessary gas overhead.
{% endstep %}
{% endstepper %}

## References

* Target file: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
