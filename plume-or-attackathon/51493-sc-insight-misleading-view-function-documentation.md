# 51493 sc insight misleading view function documentation

Submitted on Aug 3rd 2025 at 11:20:30 UTC by @heavyw8t for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #51493
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>
* Impacts: (none specified)

{% hint style="info" %}
Brief/Intro

The `_earned` function in `RewardsFacet.sol` is documented as an "Internal View Function" but actually modifies state by calling `calculateRewardsWithCheckpoints()`, which updates storage variables.
{% endhint %}

## Vulnerability Details

See POC

## Impact Details

Insight: The documentation is incorrect and misleading.

## Proof of Concept

<details>

<summary>Proof of Concept — details</summary>

The function comment states "Internal View Function (earned)," but the implementation calls `PlumeRewardLogic.calculateRewardsWithCheckpoints()`, which in turn calls `updateRewardPerTokenForValidator()`. This function modifies several storage variables:

* `$.validatorLastUpdateTimes[validatorId][token] = block.timestamp`
* `$.validatorRewardPerTokenCumulative[validatorId][token] += rewardPerTokenIncrease`
* `$.validatorAccruedCommission[validatorId][token] += commissionDeltaForValidator`

</details>
