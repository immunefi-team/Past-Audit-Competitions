# 52277 sc low race condition in streak calculation leads to unfair streak reset for users spinning near utc day change

* **Submitted on:** Aug 9th 2025 at 11:19:27 UTC by @Orionn for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #52277
* **Report Type:** Smart Contract
* **Severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

{% hint style="warning" %}
A race condition in streak calculation can unfairly reset a user's daily streak when the oracle callback is mined after a UTC day boundary, even if the user initiated their spin before the boundary.
{% endhint %}

## Description

### Brief / Intro

A race condition exists in the daily streak calculation logic within `Spin.sol`. The `_computeStreak` function uses the `block.timestamp` from the oracle's callback transaction (`handleRandomness`) to determine the day of the spin, rather than the `block.timestamp` of the user's initial `startSpin` transaction. If a user starts a spin shortly before a UTC day change, and the oracle callback is mined after the day change, the user's streak can be unfairly reset.

## Vulnerability Details

The daily streak mechanic relies on comparing the day of the last spin (`lastDaySpun`) with the day of the current spin (`today`). The contract treats the oracle callback time as the spin time, which is asynchronous and outside the user's control.

* User's Transaction (`startSpin`): user initiates spin at timestamp A (the user's intended spin time).
* Oracle's Transaction (`handleRandomness`): oracle callback occurs at timestamp B (potentially later).
* Root Cause: `_computeStreak` is called in `handleRandomness` and uses `nowTs` (the callback's block.timestamp) to compute `today`. If A and B fall on different UTC days, the streak calculation can be incorrect and penalize the user.

Vulnerable code excerpt: `Spin::_computeStreak`

```solidity
uint256 lastDaySpun = lastSpinTs / SECONDS_PER_DAY;
uint256 today = nowTs / SECONDS_PER_DAY; // `nowTs` is the block.timestamp of the handleRandomness callback
if (today == lastDaySpun + 1) { // This check fails if `today` is `lastDaySpun + 2` due to the timing gap
    return userData[user].streakCount + streakAdjustment;
}
return 0 + streakAdjustment; // The user's streak is unfairly reset
```

## Impact Details

* Loss of Value: Users can lose their accumulated daily streak. Higher streaks yield higher "Raffle Ticket" rewards (`baseRaffleMultiplier * streakForReward`), so breaking a streak reduces future rewards.
* Loss of Opportunity: Users may lose eligibility for weekly Jackpot rewards that require a minimum `streakCount`.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol?utm\\_source=immunefi>

## Proof of Concept

{% stepper %}
{% step %}

### Scenario setup

* Actor: Alice, with `streakCount = 9` and `lastSpinTimestamp` recorded on Day 9.
* Goal: Perform daily spin on Day 10 to reach `streakCount = 10`.
  {% endstep %}

{% step %}

### Step 1 — Alice starts spin just before UTC day change

* Time: Day 10, 23:59:50 UTC.
* Action: Alice calls `startSpin()`. Her transaction is mined in a block stamped on Day 10.
* From Alice's perspective, she completed the daily action for Day 10.
  {% endstep %}

{% step %}

### Step 2 — Oracle delay

* A small network/oracle delay occurs before the oracle processes the randomness request.
  {% endstep %}

{% step %}

### Step 3 — Oracle callback after UTC day change

* Time: Day 11, 00:00:10 UTC.
* The oracle's callback transaction is mined with a block timestamp on Day 11 and `handleRandomness` is executed.
  {% endstep %}

{% step %}

### Step 4 — Bug triggers in streak computation

* `_computeStreak` is called inside `handleRandomness` using `nowTs` from the callback (Day 11).
* Calculations:
  * `lastDaySpun` derived from Alice's `lastSpinTimestamp` → Day 9.
  * `today` derived from `nowTs` → Day 11.
  * Check `if (today == lastDaySpun + 1)` => checks `11 == 9 + 1` → false.
* Result: The function treats the spin as breaking the streak and returns `1`.
  {% endstep %}

{% step %}

### Final result

* `userData[alice].streakCount` is set to `1` instead of `10`. Alice's streak is lost due to an oracle timing race that is outside her control.
  {% endstep %}
  {% endstepper %}
