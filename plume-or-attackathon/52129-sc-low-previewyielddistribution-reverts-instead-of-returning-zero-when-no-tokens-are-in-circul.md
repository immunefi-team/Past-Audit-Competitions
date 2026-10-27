# 52129 sc low previewyielddistribution reverts instead of returning zero when no tokens are in circulation

**Submitted on Aug 8th 2025 at 07:49:22 UTC by @Afriauditor for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52129
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `previewYieldDistribution` function in `ArcToken.sol` reverts when the token supply is zero, instead of returning zero amounts for all holders. This unnecessarily blocks normal edge cases and can cause integrations to fail when previewing yield.

### Vulnerability Details

Inside `previewYieldDistribution`, the code explicitly checks:

```solidity
if (supply == 0) {
    revert NoTokensInCirculation();
}
```

The correct behavior should be to return arrays of holders and amounts where each amount is 0 if the total supply is 0. This ensures that preview calls work in all states and avoids breaking logic that rely on the function.

### Impact Details

Contract fails to deliver promised returns in scenarios where preview functionality is relied upon to return a value.

## Proof of Concept

{% stepper %}
{% step %}

### Setup

Have `ArcToken` in a state where `totalSupply() == 0` (e.g., no mints yet or everything burned).
{% endstep %}

{% step %}

### Action

Call `previewYieldDistribution(amount)` with any non-zero amount.
{% endstep %}

{% step %}

### Observed Behavior

The function reads `supply = totalSupply()` and hits:

```solidity
if (supply == 0) {
    revert NoTokensInCirculation();
}
```

The call reverts instead of returning arrays filled with zeros.
{% endstep %}
{% endstepper %}
