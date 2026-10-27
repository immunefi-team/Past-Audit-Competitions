# 50860 sc high logic error in jackpot eligibility check leads to systematic theft of user rewards

**Submitted on Jul 29th 2025 at 07:32:00 UTC by @AlertBasilisk56249 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50860
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:** Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

{% hint style="danger" %}
Critical logic flaw allows legitimate jackpot winners to be denied rewards due to use of stale stored streak data instead of the freshly computed streak.
{% endhint %}

## Description

The `Spin.sol` contract contains a logic flaw in `handleRandomness()` that systematically denies users legitimate jackpot rewards (5,000–100,000 PLUME). The function computes the user's current streak correctly for the current spin but uses the old stored streak value when checking jackpot eligibility, and only updates the stored streak after reward determination. As a result, users who should receive jackpots based on their actual streak receive "Nothing" instead.

Root cause summary:

* `currentSpinStreak` is computed correctly.
* Eligibility check uses `userDataStorage.streakCount` (old stored value).
* `userDataStorage.streakCount` is updated only after reward selection.

Affected lines (as reported): lines 172, 181–188, and 208 in `handleRandomness()`.

Why critical:

* Affects any user who broke their streak and returned to play.
* Jackpot amounts are large (5,000–100,000 PLUME).
* VRF randomness can indicate a jackpot win, but eligibility check can wrongly deny payout.

Economic impact (week → required streak → award):

* Week 0: streak ≥ 2 → 5,000 PLUME
* Week 2: streak ≥ 4 → 10,000 PLUME
* Week 11: streak ≥ 13 → 100,000 PLUME

## Proof of Concept

{% stepper %}
{% step %}

### Setup Phase

* User creates an account and spins daily.
* User builds streak to 3 (spins for 3 consecutive days). Contract storage: `streakCount = 3`.
* User stops spinning for 5 days (streak broken; storage still shows 3).
  {% endstep %}

{% step %}

### Exploitation Phase

* Campaign reaches week 2 (requires streak ≥ 4 for jackpot eligibility).
* User returns and calls `startSpin()` — payment accepted, VRF request initiated.
* Supra VRF returns a random number landing in the jackpot probability range.
* `handleRandomness()` is invoked with the winning randomness.
  {% endstep %}

{% step %}

### Bug Execution (detailed)

* Line 172: `_computeStreak(user, block.timestamp, true)` computes `currentSpinStreak = 1` (broken streak plus current spin).
* Lines 175–176: `determineReward()` correctly categorizes the spin as a "Jackpot" based on VRF randomness.
* Lines 181–188: Eligibility check incorrectly uses `userDataStorage.streakCount` (which is still 3) instead of `currentSpinStreak` (1).
  * Example check: `3 < (2 + 2)` → `3 < 4` evaluates true → fails eligibility.
* As a result, `rewardCategory = "Nothing"` and `rewardAmount = 0`.
* Line 208: `userDataStorage.streakCount` is finally updated to `1`, but the reward decision has already been made.

Result: User who should have received a 10,000 PLUME jackpot receives nothing due to use of stale stored streak.
{% endstep %}
{% endstepper %}

## Attack Variations

* Any user with stored streak lower than required weekly threshold is affected.
* Severity increases in later weeks (higher streak thresholds up to 13).

## Code Fix Required

Change the eligibility check to use the freshly computed streak instead of the stored value.

Replace:

```solidity
} else if (userDataStorage.streakCount < (currentWeek + 2)) {
```

With:

```solidity
} else if (currentSpinStreak < (currentWeek + 2)) {
```

This ensures eligibility uses the current calculated streak rather than outdated stored data.

## Technical Details / Location

* Function: `handleRandomness()` in Spin.sol
* Relevant lines (as reported): 172, 181–188, 208
* Affected logic: Jackpot eligibility verification uses stale stored streak count

## Impact Summary

* Direct denial of legitimate jackpot rewards (5k–100k PLUME).
* Systematic loss of user funds for broken-and-returning players.
* High-severity economic impact across users and campaign weeks.

## References

* Target file: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
