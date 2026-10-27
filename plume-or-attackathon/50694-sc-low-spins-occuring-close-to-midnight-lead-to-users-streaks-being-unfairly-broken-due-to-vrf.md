# 50694 sc low spins occuring close to midnight lead to users streaks being unfairly broken due to vrf callback delay

**Submitted on Jul 27th 2025 at 15:52:29 UTC by @heavyw8t for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50694
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The Spin contract contains a timing vulnerability where users can lose their daily streaks due to VRF callback delays. When a user initiates a spin near midnight, the VRF callback may execute after the day boundary, causing the streak calculation to incorrectly reset the user's streak to 1.

### Vulnerability Details

The issue stems from the separation of streak validation and streak calculation across two different transactions with different timestamps. The callback function `handleRandomness()` — which is called after a delay by the VRF Oracle — actually calculates if the streak has been broken, using the timestamp when the callback occurred, not the timestamp of when the user called the `spin()` function. Since the delay between calling `spin()` and the callback occurring is not influenced in any way by the user, this can lead to the user losing their streak unfairly and therefore losing yield as they will now be ineligible for the jackpot and receive less raffle rewards.

In `startSpin()`, the `canSpin` modifier only validates that the user hasn't already spun today:

```solidity
// plume/src/spin/Spin.sol:146-164
modifier canSpin() {
    // Early return if the user is whitelisted
    if (whitelists[msg.sender]) {
        _;
        return;
    }

    UserData storage userDataStorage = userData[msg.sender];
    uint256 _lastSpinTimestamp = userDataStorage.lastSpinTimestamp;

    // Retrieve last spin date components
    (uint16 lastSpinYear, uint8 lastSpinMonth, uint8 lastSpinDay) = (
        dateTime.getYear(_lastSpinTimestamp),
        dateTime.getMonth(_lastSpinTimestamp),
        dateTime.getDay(_lastSpinTimestamp)
    );

    // Retrieve current date components
    (uint16 currentYear, uint8 currentMonth, uint8 currentDay) =
        (dateTime.getYear(block.timestamp), dateTime.getMonth(block.timestamp), dateTime.getDay(block.timestamp));

    // Ensure the user hasn't already spun today
    if (isSameDay(lastSpinYear, lastSpinMonth, lastSpinDay, currentYear, currentMonth, currentDay)) {
        revert AlreadySpunToday();
    }

    _;
}
```

However, in `handleRandomness()`, the actual streak calculation uses a different timestamp:

```solidity
// plume/src/spin/Spin.sol:207
uint256 currentSpinStreak = _computeStreak(user, block.timestamp, true);
```

The `_computeStreak()` function determines streak continuation using simple day arithmetic:

```solidity
// plume/src/spin/Spin.sol:306-324
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

## Impact Details

* Streak Loss: Users lose accumulated daily streaks despite properly initiating spins on consecutive days.
* Jackpot Eligibility: Users may become ineligible for jackpot claims due to artificially broken streaks.
* Fewer Raffle rewards: Users will receive less raffle rewards as their multiplier is now smaller.

## Proof of Concept

{% stepper %}
{% step %}

### Setup

User has an active 5-day streak (last spin on day C).
{% endstep %}

{% step %}

### Step: Spin near midnight

Day D at 11:59 PM: User calls `startSpin()`.\
Oracle request is initiated. Spin is marked as pending.
{% endstep %}

{% step %}

### Step: VRF callback after midnight

Day E at 12:01 AM: Oracle calls `handleRandomness()`.\
`_computeStreak()` uses day E timestamp:

* lastDaySpun = C
* today = E
* Since E > C + 1, condition fails
* Streak resets to 1 instead of continuing to 6
  {% endstep %}
  {% endstepper %}

Result: User loses 5-day streak despite valid consecutive daily participation.

## References

<details>

<summary>Relevant links</summary>

* Target source: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>

</details>
