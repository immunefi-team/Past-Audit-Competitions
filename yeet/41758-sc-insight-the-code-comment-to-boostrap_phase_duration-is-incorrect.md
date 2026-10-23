# #41758 \[SC-Insight] The code comment to \`BOOSTRAP\_PHASE\_DURATION\` is incorrect

**Submitted on Mar 18th 2025 at 06:20:12 UTC by @h2134 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41758
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeet.sol>
* **Impacts:**

## Description

## Brief/Intro

The code comment to `BOOSTRAP_PHASE_DURATION` is incorrect.

## Vulnerability Details

The code comment to `BOOSTRAP_PHASE_DURATION` is as below:

```solidity
    uint256 public BOOSTRAP_PHASE_DURATION; // The time between rounds
```

This comment is incorrect as `BOOSTRAP_PHASE_DURATION` is the duration of bootstrap phrase, not the time between rounds.

We believe the code comment is wrongly copied from `COOLDOWN_TIME`.

```solidity
    uint256 public COOLDOWN_TIME; // The time between rounds
```

## Impact Details

Incorrect documentation.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeet.sol#L106>

## Proof of Concept

## Proof of Concept

This comment is incorrect as `BOOSTRAP_PHASE_DURATION` is the duration of bootstrap phrase, not the time between rounds.
