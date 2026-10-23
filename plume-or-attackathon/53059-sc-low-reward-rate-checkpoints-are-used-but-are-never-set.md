# 53059 sc low reward rate checkpoints are used but are never set

**Submitted on Aug 14th 2025 at 18:30:04 UTC by @a16 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53059
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

Several functions use the `rewardRateCheckpoints[]` mapping, but checkpoints are never set for a specific token — only for a token/validator pair.

### Vulnerability Details

Functions like `getRewardRateCheckpoint()` and `getRewardRateCheckpointCount()` use `rewardRateCheckpoints[token]`, but `rewardRateCheckpoints[token].length` is always `0` as it is never set.

Example snippets from the codebase that demonstrate the issue:

```solidity
if (index >= $.rewardRateCheckpoints[token].length) {
    revert InvalidRewardRateCheckpoint(token, index);
}
```

Since `rewardRateCheckpoints[token].length` is always `0`, this is effectively:

```solidity
if (index >= 0) {
    revert InvalidRewardRateCheckpoint(token, index);
}
```

So `getRewardRateCheckpointCount()` will always return `0`, and `getRewardRateCheckpoint()` will always revert.

### Impact Details

* Users calling these functions will either receive `0` for checkpoint counts or encounter a revert when attempting to read a checkpoint.
* This results in functions failing to deliver the promised returns information (or related behavior) but does not directly cause loss of funds or contract value.

## Suggestions

`rewardRateCheckpoints[]` appears to be vestigial (unused) or misused. Consider one of the following:

* Remove `rewardRateCheckpoints` and all references to it if it is truly unused.
* Or, if intended, ensure checkpoints are populated for tokens (not only token/validator pairs) and update the accessors to reference the correct storage structure.

## Proof of Concept

{% stepper %}
{% step %}

### Reproduce

1. Call `getRewardRateCheckpoint()` for a token and any index.

Result: the call reverts due to the check against an always-zero length.
{% endstep %}
{% endstepper %}
