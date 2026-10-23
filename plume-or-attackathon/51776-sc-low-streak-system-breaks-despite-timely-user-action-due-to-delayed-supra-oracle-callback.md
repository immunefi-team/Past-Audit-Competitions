# 51776 sc low streak system breaks despite timely user action due to delayed supra oracle callback

* Submitted on Aug 5th 2025 at 18:36:20 UTC by @light279 for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* Report ID: #51776
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* Impacts:
  * Temporary freezing of funds for at least 24 hours
  * Protocol insolvency

## Description

### Brief/Intro

The contract allows users to spin once per day and maintains a streak system to reward consecutive daily spins. However, the streak computation relies on `block.timestamp` inside the `Spin::handleRandomness` callback, which is triggered asynchronously by the Supra Oracle. This introduces a time gap between the user-initiated `Spin::startSpin` and the actual randomness handling, potentially breaking user streaks even if the user called the function before the end of the day.

### Vulnerability Details

The function `_computeStreak` calculates whether a user’s spin continues their streak based on the current block timestamp during the `handleRandomness` execution. But since `startSpin` only emits a randomness request and the actual response is handled asynchronously (likely in a different block and time), a delay in the Supra oracle’s callback may push the effective time of `handleRandomness` into the next day.

This results in `nowTs / SECONDS_PER_DAY != lastSpinTs / SECONDS_PER_DAY + 1` even when the user called `startSpin` on time. The streak is then reset despite correct user behavior.

Code excerpt of `_computeStreak`:

{% code title="Spin.sol — \_computeStreak" %}

```
```

{% endcode %}

```javascript
function _computeStreak(address user, uint256 nowTs, bool justSpun) internal view returns (uint256) {
        // if a user just spun, we need to increment the streak its a new day or a broken streak
        uint256 streakAdjustment = justSpun ? 1 : 0;

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
    }
```

This issue manifests in `handleRandomness` where `block.timestamp` (the oracle callback time) is used instead of the user's original spin time:

{% code title="Spin.sol — handleRandomness (excerpt)" %}

```
```

{% endcode %}

```javascript
 function handleRandomness(uint256 nonce, uint256[] memory rngList) external onlyRole(SUPRA_ROLE) nonReentrant {
        address payable user = userNonce[nonce];
        if (user == address(0)) {
            revert InvalidNonce();
        }

        isSpinPending[user] = false;
        delete userNonce[nonce];
        delete pendingNonce[user];

        uint256 currentSpinStreak = _computeStreak(user, block.timestamp, true);
        uint256 randomness = rngList[0]; // Use full VRF range
        
............................
```

Using `block.timestamp` here does not reflect the user's original spin time, but the time at which the oracle responds.

### Impact Details

{% hint style="warning" %}

* Loss of Streaks: Users who spin on time may still lose their streak due to oracle delay.
* Frustration: This creates a poor UX where users appear to have done everything correctly, but are penalized due to backend timing.
  {% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Scenario setup

Assume current day is Day X. A user spins at 23:59 (end of Day X) by calling `startSpin()`.
{% endstep %}

{% step %}

### Oracle latency occurs

`startSpin()` correctly generates a Supra oracle request and passes callback signature `handleRandomness(uint256,uint256[])`. Due to Oracle latency, the callback `handleRandomness()` is only called at `00:00:10` on Day `X+1`.
{% endstep %}

{% step %}

### Streak calculation at callback

In `handleRandomness()`, the `_computeStreak()` function is called with `block.timestamp = 00:00:10`, which translates to Day `X+1`. `_computeStreak()` checks:

```javascript
if (today == lastDaySpun + 1) {
    return userData[user].streakCount + 1;
}
```

But `today = X+1`, `lastDaySpun = X-1`, so the condition fails.
{% endstep %}

{% step %}

### Result

The fallback clause is executed:

```javascript
return 0 + 1;
```

So the user's streak is reset even though they spun on Day X. This causes unexpected behavior and harms streak-based rewards.
{% endstep %}
{% endstepper %}
