# 52286 sc high off by one error in jackpot eligibility check leads to denial of legitimate rewards

**Submitted on Aug 9th 2025 at 14:04:30 UTC by @OxPrince for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52286
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The Spin contract contains a critical off-by-one error in its jackpot eligibility verification logic where it uses stale pre-spin streak data instead of the correctly computed post-spin streak value. This bug systematically denies jackpot rewards to users who achieve the required streak exactly through their winning spin, causing legitimate winners to receive "Nothing" instead of their earned jackpot prizes. In production, this would result in unfair denial of potentially thousands of dollars worth of jackpot rewards to qualifying users.

### Vulnerability Details

The vulnerability exists in the `handleRandomness` function (Spin.sol:207-265) where there's an inconsistency between how the streak is calculated and how it's used for jackpot eligibility checks.

Problem flow summary:

* Correct post-spin streak calculation: The function properly computes the updated streak that accounts for the current spin (Spin.sol:217).
* Inconsistent jackpot eligibility check: The jackpot eligibility verification uses the stored pre-spin streak value instead of the computed post-spin value (Spin.sol:232).
* Late streak update: The streak is only updated to the correct post-spin value after all reward logic completes (Spin.sol:253).
* Inconsistent behavior with other rewards: Other rewards like raffle tickets correctly use the post-spin streak for their calculations (Spin.sol:298).

The `_computeStreak` function with `justSpun = true` correctly accounts for the current spin's contribution to the streak (Spin.sol:307-325), but this computed value is ignored for jackpot eligibility checks.

The required streak increases weekly (currentWeek + 2) as shown in the `getWeeklyJackpot` function (Spin.sol:430), making this edge case particularly impactful for users who are exactly one spin away from the requirement.

## Impact Details

* Users who legitimately qualify for jackpots through their winning spin are systematically denied their rewards.
* The jackpot funds remain in the contract rather than being distributed to rightful winners.

## References

* Spin contract implementation: plume/src/spin/Spin.sol
* Test cases demonstrating expected behavior: plume/test/Spin.t.sol
* Spin system documentation: plume/SPIN.md
* Target repo (file link above): <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>

## Proof of Concept

{% stepper %}
{% step %}

### PoC — Step

User has streak count of 1.
{% endstep %}

{% step %}

### PoC — Step

User spins and gets jackpot-winning randomness (e.g., randomness = 0).
{% endstep %}

{% step %}

### PoC — Step

`handleRandomness` is called:

* `currentSpinStreak` is correctly computed as 2 (accounting for current spin).
* `determineReward` returns ("Jackpot", jackpotAmount) based on randomness.
* Jackpot eligibility check uses `userDataStorage.streakCount` (value: 1) instead of `currentSpinStreak` (value: 2).
* Check fails: `1 < (0 + 2)` evaluates to true.
* Reward is downgraded to "Nothing" and `NotEnoughStreak` event is emitted.
  {% endstep %}

{% step %}

### PoC — Step

User's streak is updated to 2 after the reward logic, but it's too late.

Expected behavior: User should receive the jackpot since their post-spin streak (2) meets the requirement (2).

Actual behavior: User receives "Nothing" due to the stale pre-spin streak value being used in the eligibility check.

This edge case is not covered by the test `testJackpotInsufficientThenSufficientStreak` (Spin.t.sol:643-697), which checks multi-day progression but not the case where the qualifying streak is achieved on the same spin as the jackpot win.
{% endstep %}
{% endstepper %}

## Recommendation

Replace the pre-spin streak reference with the computed post-spin streak in the jackpot eligibility check. Specifically, change the condition in `handleRandomness` to use `currentSpinStreak` instead of `userDataStorage.streakCount`:

{% code title="Spin.sol — suggested change" %}

```solidity
} else if (currentSpinStreak < (currentWeek + 2)) {
```

{% endcode %}

This ensures jackpot eligibility uses the same post-spin streak value as other reward calculations, maintaining consistency and fairness.

{% hint style="warning" %}
Do not change the required streak logic—only ensure the eligibility check uses the up-to-date streak computed for the current spin.
{% endhint %}
