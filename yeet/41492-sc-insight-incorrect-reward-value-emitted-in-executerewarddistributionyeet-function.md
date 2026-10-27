# #41492 \[SC-Insight] Incorrect Reward Value Emitted in \`executeRewardDistributionYeet\` Function

**Submitted on Mar 15th 2025 at 21:16:25 UTC by @chista0x for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41492
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In the `executeRewardDistributionYeet` function, the event `RewardsDistributedToken0` is emitted with `accRevToken0` as its parameter, which represents the total accumulated rewards rather than the actual amount distributed. This misrepresentation can lead to inaccurate off-chain tracking and analytics.

## Vulnerability Details

The function calculates the accumulated rewards (`accRevToken0`) and ensures that the swap's `inputAmount` does not exceed this total. However, when approving tokens and executing the swap, only `swap.inputAmount` is actually intended to be used for the distribution. Despite this, the event is emitted with `accRevToken0`:

```solidity
emit RewardsDistributedToken0(accRevToken0, rewardIndex);
```

This means that the emitted event logs a value that does not accurately reflect the amount of tokens that were actually processed through the swap. As a result, any external monitoring systems or analytics relying on these event logs will have misleading data regarding reward distribution.

## Impact Details

* **Misleading Event Data:** The event log reports an inflated reward amount, which could confuse off-chain analytics, user interfaces, or other monitoring tools.
* **Inaccurate Accounting:** Relying on these events for auditing purposes or reward calculations may lead to discrepancies in tracking the actual funds distributed.
* **Reduced Transparency:** The discrepancy undermines the transparency of the reward distribution process, potentially affecting stakeholder trust.

## Recommendation

To ensure that the event accurately reflects the actual tokens used for the reward distribution, update the event emission to use `swap.inputAmount` instead of `accRevToken0`. The modified code should be:

```solidity
emit RewardsDistributedToken0(swap.inputAmount, rewardIndex);
```

This adjustment will provide a correct representation of the reward distribution and improve off-chain monitoring and auditing processes.

## References:

[StakeV2.sol](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L179)

## Proof of Concept

## Proof of Concept (POC)

The core issue can be observed in the following snippet:

```solidity
uint256 accRevToken0 = accumulatedDeptRewardsYeet();
// ... token approval and swap execution using swap.inputAmount ...
emit RewardsDistributedToken0(accRevToken0, rewardIndex);
```

Here, although `swap.inputAmount` is the correct amount intended for the distribution, `accRevToken0` (which could be greater) is being emitted, leading to a mismatch in the reported reward amount.
