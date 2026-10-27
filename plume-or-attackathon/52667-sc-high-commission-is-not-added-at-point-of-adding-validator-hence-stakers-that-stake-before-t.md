# 52667 sc high commission is not added at point of adding validator hence stakers that stake before the first checkpoint would always use the current commission&#x20;

**Submitted on Aug 12th 2025 at 10:58:35 UTC by @silver\_eth for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52667
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief / Intro

When a new validator is added, a commission checkpoint is not pushed. During the period before a commission checkpoint exists, commission is always calculated as the current commission. As a result, stakers who staked before the first commission checkpoint can end up accruals computed with a different commission than the validator actually accrued, creating discrepancies that can lead to a shortfall or unfair loss of rewards.

### Vulnerability Details

Factors that make this issue possible:

* When a new validator is added, a commission checkpoint is not pushed; only a reward rate checkpoint is pushed.
* In PlumeRewardLogic, when there is no checkpoint for the selected past time, the code returns the current commission rate: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L620-L622>
* User rewards are not accrued at the same time as the validator commission.

Consequence: users who stake before the first commission checkpoint will have their commissions for that pre-checkpoint period calculated using the current commission rate (which may differ from the initial commission). For example:

* If the initial commission when the validator was added was 20%, but the current commission later is 10%, the validator will have rightly accrued 20% of total rewards for that earlier period, but users updating/claiming afterward will be charged only 10% — creating a deficit the treasury must cover.
* Conversely, if the current commission is greater than the initial (e.g., initial 20% → current 30%), stakers could be overcharged for the prior period, losing rewards improperly.

### Impact Details

* Protocol insolvency risk if the current commission is smaller than the initial (treasury may be insufficient to cover validator's rightful commission).
* Loss of rewards for stakers if the current commission is larger than the initial.
* A validator (or a colluding party) could exploit timing to create or benefit from the discrepancy.

{% hint style="warning" %}
The issue affects both on-chain accounting and user-facing reward calculations and can be exploited by validators or administrators who change commissions after stakes/rewards accrue but before stakers' states are updated.
{% endhint %}

## References

<details>

<summary>Relevant code references</summary>

* PlumeRewardLogic commission lookup (example lines):\
  <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L620-L622>
* Additional reference mention in report:\
  <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L477-L482>

</details>

## Proof of Concept

{% stepper %}
{% step %}

### Step

Protocol adds a new validator with commission = 20%.
{% endstep %}

{% step %}

### Step

Bob stakes to this validator totalling 100\_000 plume and some rewards accrue; for this example, 1000 plume.
{% endstep %}

{% step %}

### Step

Validator admin sets new commission to 10%. This settles the validator commission for the previous time period at 20% — i.e., 200 plume is recorded as the validator's commission.
{% endstep %}

{% step %}

### Step

Bob claims his rewards in the same block but after the commission is changed. Because the current commission is now 10%, Bob receives 900 plume.
{% endstep %}

{% step %}

### Step

Validator admin requests and claims their 200 plume, leaving the treasury with a 100 plume deficit.
{% endstep %}
{% endstepper %}
