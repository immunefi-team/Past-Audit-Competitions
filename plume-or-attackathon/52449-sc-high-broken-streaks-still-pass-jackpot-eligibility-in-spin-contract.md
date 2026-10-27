# 52449 sc high broken streaks still pass jackpot eligibility in spin contract

**Submitted on Aug 10th 2025 at 19:25:14 UTC by @farman1094 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52449
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

The Spin contract contains a broken logic flaw in its jackpot eligibility check: it verifies a user's "streak count" using outdated data instead of the actual current streak. As a result, users who have broken their streak (i.e., missed spins for one or more days) may still be able to claim the jackpot reward, violating the intended requirement that only users with an active, consecutive streak are eligible.

### Vulnerability Details

In the Spin contract's `handleRandomness` function, the eligibility to claim a jackpot is checked using the user's stored streak count (`userDataStorage.streakCount`) before it is updated:

```solidity
else if (userDataStorage.streakCount < (currentWeek + 2)) {
    // Not enough streak count to claim Jackpot
    ...
}
```

However, the actual current streak has already been calculated based on the user's last spin timestamp and the present day, using the `_computeStreak` function, but not used.

```solidity
uint256 currentSpinStreak = _computeStreak(user, block.timestamp, true);
```

If a user has not spun for several days, their streak should be considered broken and reset to one. But the check above compares the old, stored value, which may still be high if the user previously had a long streak. This allows users with expired streaks to pass the jackpot eligibility check, even though their streak is not consecutive.

{% hint style="warning" %}
Impact: This vulnerability allows users to claim jackpot rewards without meeting the intended requirements. The integrity of the reward system is compromised, potentially leading to significant financial loss as rewards are claimed by users who shouldn't be eligible.
{% endhint %}

## Solution

We can use the current computed streak for the comparison:

```
uint256 currentSpinStreak = _computeStreak(user, block.timestamp, true);
```

If we do not want to consider this current spin to add in streak we can subtract one and use that for comparison.

{% hint style="info" %}
Use the computed currentSpinStreak (optionally adjusted by -1 if the current spin should not be counted) instead of the stored `userDataStorage.streakCount` when checking jackpot eligibility.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Build a streak

User builds up a streak count higher than the required threshold (e.g., 10).
{% endstep %}

{% step %}

### Stop spinning

User stops spinning for several days (streak should reset).
{% endstep %}

{% step %}

### Old state remains

The old streak value remains in storage not updated yet.
{% endstep %}

{% step %}

### Re-spin and compute current streak

When spinning again and hitting a jackpot, the contract computes the streak which should be 0 or 1 now.

```solidity
// _computeStreak
uint256 lastSpinTs = userData[user].lastSpinTimestamp;

if (lastSpinTs == 0) {
    return 0 + streakAdjustment;
}
uint256 lastDaySpun = lastSpinTs / SECONDS_PER_DAY;
uint256 today = nowTs / SECONDS_PER_DAY;
if (today == lastDaySpun) {
    return userData[user].streakCount;
} // same day
if (today == lastDaySpun + 1) {
    return userData[user].streakCount + streakAdjustment;
} // streak not broken yet
return 0 + streakAdjustment; // broken streak
```

{% endstep %}

{% step %}

### Incorrect eligibility check

But in the comparison of streak for jackpot checks it checks with old state instead of the updated one, which could allow the user to claim the jackpot despite an expired streak:

```solidity
// handleRandomness
} else if (userDataStorage.streakCount < (currentWeek + 2)) {
    // Not enough streak count to claim Jackpot
}
```

{% endstep %}
{% endstepper %}
