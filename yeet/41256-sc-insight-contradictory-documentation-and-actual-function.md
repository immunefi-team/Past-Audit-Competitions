# #41256 \[SC-Insight] Contradictory Documentation and actual function

**Submitted on Mar 13th 2025 at 03:27:02 UTC by @xdead4f for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41256
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/RewardSettings.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The RewardSettings.sol contract contains critical inconsistencies between variable names, comments, and actual implementation logic, creating significant risk of misunderstanding by developers, auditors, and users. This misconception could lead to unexpected reward distribution behavior and potential exploitation.

## Vulnerability Details

RewardSettings.sol: Variable `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR`\
Reward.sol: Integration with the above variable in reward calculation logic

The variable `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` has a comment stating:

```
constructor() Ownable(msg.sender) {
        /// @dev this is in percentage, 1/10 of the total rewards
        MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR = 30;
    }
```

This comment suggests:

1. The value represents a percentage
2. It should equate to "1/10" of total rewards
3. But is initialized to 30

These statements are internally inconsistent and don't align with how the value is actually used.

## Impact Details

1. Unintended Reward Distribution: If an admin intends to increase the reward cap by increasing the parameter value (based on its name), they would actually be decreasing the maximum reward.
2. Whale Exploitation Risk: If mistakenly set too low (e.g., 2 instead of 20), a single wallet could claim up to 50% of epoch rewards, potentially draining the reward pool unfairly.

## Proof of Concept

Given this scenario :

1. Current setting: 30 → Max reward = \~3.33% of epoch rewards
2. Admin wants to double maximum rewards
3. Misled by documentation, changes to 60
4. Actual result: Max reward = \~1.67% (halved instead of doubled)
