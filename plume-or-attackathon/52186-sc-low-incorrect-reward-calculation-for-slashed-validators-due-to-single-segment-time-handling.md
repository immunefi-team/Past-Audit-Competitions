# 52186 sc low incorrect reward calculation for slashed validators due to single segment time handling&#x20;

**Submitted on Aug 8th 2025 at 15:20:58 UTC by @DSbeX for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52186
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

The `calculateRewardsWithCheckpoints` function contains a vulnerability in its slashed validator reward calculation logic. By using a single time segment with a static reward rate between the last update and slash time, it ignores intermediate reward rate changes. This results in reward miscalculations that would cause financial losses to users (through underpayment) or protocol drains (through overpayment) if exploited in production.

### Vulnerability Details

The vulnerability is in the slashed validator handling within `calculateRewardsWithCheckpoints`. When a validator is slashed, rewards should be calculated up to the slash timestamp, accounting for all reward rate changes during this period. However, the current implementation uses a single continuous segment with a static rate:

```javascript
} else {
    // Slashed validator case
    uint256 currentCumulativeRewardPerToken = ...;
    uint256 effectiveEndTime = validator.slashedAtTimestamp;
    
    if (effectiveEndTime > validatorLastUpdateTime) {
        uint256 timeSinceLastUpdate = effectiveEndTime - validatorLastUpdateTime;
        uint256 effectiveRewardRate = 
            getEffectiveRewardRateAt($, token, validatorId, validatorLastUpdateTime).rate;
        
        //Vulnerable single-segment calculation
        uint256 rewardPerTokenIncrease = timeSinceLastUpdate * effectiveRewardRate;
        currentCumulativeRewardPerToken += rewardPerTokenIncrease;
    }
    return _calculateRewardsCore(...);
}
```

* Uses one continuous time segment from `validatorLastUpdateTime` to `effectiveEndTime`
* Static rate application: Applies the initial reward rate at `validatorLastUpdateTime` for the entire duration
* Checkpoint ignorance: Fails to account for intermediate reward rate changes captured in `validatorRewardRateCheckpoints`

### Impact Details

{% hint style="warning" %}
Reward-rate changes during unprocessed periods can cause large underpayments to users or overpayments by the protocol. Because slashed-validator calculations are final, those errors are permanent and compound as delayed slashes or repeated rate changes amplify the discrepancy. Miscalculated base rewards also corrupt validator commission accounting, creating a stealth drain that looks like minor rounding but adds up.
{% endhint %}

## References

<details>

<summary>Relevant code references</summary>

* calculateRewardsWithCheckpoints - <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L374>
* getEffectiveRewardRateAt returns static rate instead of segmented rates - <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/lib/PlumeRewardLogic.sol#L544>

</details>

## Proof of Concept

{% stepper %}
{% step %}

### Setup

* Validator is active and earning rewards normally
* Last reward update occurred at time T0
* Reward rate at T0 is R1 (100 tokens/day)
  {% endstep %}

{% step %}

### Reward rate update (T1)

* Protocol admin updates reward rate to R2 (200 tokens/day) at T1
* New checkpoint created at T1 with R2
* Validator continues operating but hasn't had its rewards updated since T0
  {% endstep %}

{% step %}

### Validator slashing (T2)

* Validator is slashed at time T2
* `validator.slashed` flag set to true
* `validator.slashedAtTimestamp` set to T2
* Validator stops participating after T2
  {% endstep %}

{% step %}

### Reward calculation trigger

* User claims rewards after T2
* System calls `calculateRewardsWithCheckpoints()`
* Since validator is slashed, it enters the vulnerable code path
  {% endstep %}

{% step %}

### Flawed calculation

* System calculates entire period (T0 to T2) at initial rate R1
* `effectiveRewardRate = getEffectiveRewardRateAt(validatorLastUpdateTime)` → returns R1 (rate at T0)
* rewardPerTokenIncrease = (T2 - T0) \* R1
  {% endstep %}

{% step %}

### Incorrect reward distribution

* User receives rewards for T0-T2 period calculated entirely at R1

Correct calculation should be:

* T0-T1 segment: (T1 - T0) × R1
* T1-T2 segment: (T2 - T1) × R2

Impact:

* Protocol underpaying user by (T2-T1) × (R2 - R1) tokens
* Validator commission also miscalculated based on wrong base rewards
  {% endstep %}
  {% endstepper %}
