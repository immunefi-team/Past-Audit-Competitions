# 52278 sc high incorrect streak check in jackpot eligibility leads to unfair reward denial

**Submitted on Aug 9th 2025 at 11:35:36 UTC by @godwinudo for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52278
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

The Spin contract computes a user's updated streak count when they spin, but when checking if they're eligible for a jackpot reward, it uses their old streak count from storage instead of the newly calculated value. The `handleRandomness` function incorrectly checks a user’s old streak count (`userDataStorage.streakCount`) instead of the updated streak (`currentSpinStreak`) when determining jackpot eligibility. This means users whose streak becomes sufficient after their current spin are incorrectly denied jackpot rewards they should have won.

### Vulnerability Details

The `handleRandomness` function is called by the Supra oracle to finalize a spin. It calculates the user’s streak, determines the reward, and applies it.

The `_computeStreak` function calculates the user’s streak based on their last spin timestamp (`lastSpinTimestamp`) and the current timestamp (`block.timestamp`). It uses calendar days (86,400 seconds) to determine if the spin is on the same day, the next day, or after a gap:

```solidity
function _computeStreak(address user, uint256 nowTs, bool justSpun) internal view returns (uint256) {
    uint256 streakAdjustment = justSpun ? 1 : 0;
    uint256 lastSpinTs = userData[user].lastSpinTimestamp;
    if (lastSpinTs == 0) {
        return 0 + streakAdjustment;
    }
    uint256 lastDaySpun = lastSpinTs / SECONDS_PER_DAY;
    uint256 today = nowTs / SECONDS_PER_DAY;
    if (today == lastDaySpun) {
        return userData[user].streakCount;
    }
    if (today == lastDaySpun + 1) {
        return userData[user].streakCount + streakAdjustment;
    }
    return 0 + streakAdjustment;
}
```

When `justSpun = true` (as in `handleRandomness`), the function accounts for the current spin. If the spin is on the next day (`today == lastDaySpun + 1`), it increments the streak (`streakCount + 1`). If a day is missed, it resets to 1. The result, `currentSpinStreak`, represents the user’s streak after the current spin.

The jackpot eligibility check occurs if `determineReward` returns a Jackpot reward (based on the random number falling within the `jackpotProbabilities` threshold). Two conditions must be met:

* The jackpot hasn’t been claimed in the current week (`currentWeek != lastJackpotClaimWeek`).
* The user’s streak meets the requirement (`userDataStorage.streakCount >= currentWeek + 2`).

The issue is in the streak check: `userDataStorage.streakCount < (currentWeek + 2)`. This uses the streak stored before the current spin, not `currentSpinStreak`, which includes the current spin’s contribution.

After the jackpot check, `userDataStorage.streakCount` is updated to `currentSpinStreak`. This means the correct streak is available later in the function but not used for the eligibility determination.

### Impact Details

Users are unfairly denied jackpot rewards they should receive when their current spin increments their streak to meet the jackpot threshold. This undermines fairness and may reduce participation in the spin feature.

## Proof of Concept

{% stepper %}
{% step %}

### Setup

User has been spinning for consecutive days and currently has a 1-day streak stored in their `userData.streakCount`.
{% endstep %}

{% step %}

### Week & Requirement

It's Week 0 of the campaign, so jackpot requirement is `currentWeek + 2 = 0 + 2 = 2`.
{% endstep %}

{% step %}

### Spin

User spins on their second consecutive day. The `_computeStreak` function calculates their new streak as 2 (qualifying for jackpot).
{% endstep %}

{% step %}

### Randomness

Random number generator returns a value in the jackpot range for that day. `determineReward` correctly identifies this as a "Jackpot" reward using the new streak count of 2.
{% endstep %}

{% step %}

### Incorrect Eligibility Check

In the jackpot eligibility check, the code compares `userDataStorage.streakCount` (which is still 1) against the requirement of 2.
{% endstep %}

{% step %}

### Denied Reward

Since 1 < 2, the user is denied the jackpot and receives Nothing instead. After the check, the user's streak is then updated to 2 in storage, but it's too late — they've already lost their legitimate jackpot reward.
{% endstep %}
{% endstepper %}

Expected Result: User should receive the jackpot reward since their updated streak (2) meets the requirement (2).\
Actual Result: User receives "Nothing" because the check used their outdated streak (1).

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L207C1-L266C1>
