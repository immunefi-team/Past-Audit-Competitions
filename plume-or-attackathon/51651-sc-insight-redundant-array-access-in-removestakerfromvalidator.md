# 51651 sc insight redundant array access in removestakerfromvalidator

**Submitted on Aug 4th 2025 at 16:13:23 UTC by @AasifUsmani for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51651
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeValidatorLogic.sol>

## Description

Brief/Intro

In the `removeStakerFromValidator` function of the `PlumeValidatorLogic` library, an unnecessary array access occurs during the removal of a staker from a validator's list. Specifically, the declaration and access of `lastStaker` happen unconditionally, even when no swap is needed (i.e., when removing the last element). This leads to a redundant storage read (SLOAD) that can be avoided by moving the access inside the conditional that performs the swap.

## Vulnerability Details

The issue is in Part 1 of the function, which uses a swap-and-pop pattern to remove a staker from `$.validatorStakers[validatorId]`:

```solidity
if (indexToRemove < listLength && stakersList[indexToRemove] == staker) {
    address lastStaker = stakersList[listLength - 1];  // Redundant if no swap is needed
    if (indexToRemove != listLength - 1) {
        stakersList[indexToRemove] = lastStaker;
        $.userIndexInValidatorStakers[lastStaker][validatorId] = indexToRemove;
    }
    stakersList.pop();
}
```

* The line `address lastStaker = stakersList[listLength - 1];` is executed regardless of whether a swap will occur.
* If `indexToRemove == listLength - 1` (removing the last element), no swap is needed — only `pop()` is required. In that case, `lastStaker` is never used, making the read unnecessary.
* This results in a redundant SLOAD, costing gas without changing behavior.
* Functionally the code is correct; this is a low-risk optimization opportunity.

{% hint style="info" %}
Severity: Low / Informational — No security risk or incorrect state change. Optimization and clarity improvement only.
{% endhint %}

## Impact Details

* Gas Efficiency: Each redundant SLOAD adds gas (roughly 100–200 gas per call). In high-frequency removal scenarios (mass unstaking, bulk operations), this can accumulate.
* Code Quality: Removing the redundant read improves clarity and aligns with standard swap-and-pop patterns in Solidity, reducing potential confusion for maintainers and auditors.

## Recommended Fix

Move the `lastStaker` declaration inside the conditional that performs the swap so it is only read when needed:

```solidity
if (indexToRemove < listLength && stakersList[indexToRemove] == staker) {
    if (indexToRemove != listLength - 1) {
        address lastStaker = stakersList[listLength - 1];
        stakersList[indexToRemove] = lastStaker;
        $.userIndexInValidatorStakers[lastStaker][validatorId] = indexToRemove;
    }
    stakersList.pop();
}
```

This avoids the unnecessary storage read when removing the last element.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeValidatorLogic.sol#L94>

## Proof of Concept

Consider a validator's `stakersList` with 3 elements: \[0xA, 0xB, 0xC] (length = 3).

{% stepper %}
{% step %}

### Remove last element (0xC)

* indexToRemove = 2
* listLength = 3

Current code:

* Reads `lastStaker = stakersList[2]` (0xC) unconditionally.
* Checks `if (2 != 2)` → false, so no swap.
* Pops the last element → resulting list: \[0xA, 0xB]

Redundancy: `lastStaker` was read but never used.
{% endstep %}

{% step %}

### Optimized flow

With the recommended change:

* The code checks `if (indexToRemove != listLength - 1)` first.
* Since it's false, it skips reading `stakersList[listLength - 1]`.
* Calls `pop()` → resulting list: \[0xA, 0xB]

Benefit: One less SLOAD (gas saved) when removing the last element.
{% endstep %}
{% endstepper %}
