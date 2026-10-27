# 52986 sc high jackpot check uses previous streakcount instead of current computed streak denying jackpot on first eligible day

**Submitted on Aug 14th 2025 at 15:10:59 UTC by @daxun for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52986
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

{% hint style="warning" %}
High severity: Jackpot eligibility check uses stored `streakCount` (previous value) instead of the freshly computed streak that includes today's spin.
{% endhint %}

## Description

### Brief / Intro

During a spin, the contract computes:

* `currentSpinStreak = _computeStreak(user, now, true)` (which includes today’s spin)

but verifies jackpot eligibility against the stored `userData[user].streakCount` (yesterday’s value). On the first day a user reaches the required streak, they can be downgraded to “Nothing”.

### Vulnerability Details

In `handleRandomness()` (simplified):

{% code title="handleRandomness (simplified)" %}

```solidity
uint256 currentSpinStreak = _computeStreak(user, block.timestamp, true);
...
if (isJackpot) {
    uint256 currentWeek = getCurrentWeek();
    if (currentWeek == lastJackpotClaimWeek) { ... }
    else if (userDataStorage.streakCount < (currentWeek + 2)) {
        // ↓ uses previous stored streak instead of currentSpinStreak
        rewardCategory = "Nothing"; rewardAmount = 0;
    } else {
        // award jackpot
    }
}
...
userDataStorage.streakCount = currentSpinStreak; // updated only after the check
```

{% endcode %}

Because `userDataStorage.streakCount` is updated **after** the check, a user who **just** reached the required streak **today** is unfairly denied.

## Impact Details

* **Immunefi Impact:** Low — Contract fails to deliver promised returns, but doesn't lose value
* Users meeting the milestone can miss the jackpot exactly on the day it should be awarded.

## Proof of Concept

{% stepper %}
{% step %}

### Scenario setup

Assume `requiredStreak = currentWeek + 2 = 5`.
{% endstep %}

{% step %}

### Yesterday's stored streak

Yesterday’s stored `userData[user].streakCount == 4` (consecutive).
{% endstep %}

{% step %}

### Today's computed streak

Today, `_computeStreak(..., true)` returns `5`.
{% endstep %}

{% step %}

### Jackpot check uses stored value

Jackpot path checks `userDataStorage.streakCount < 5` → **true**, so it sets reward to **“Nothing”**.
{% endstep %}

{% step %}

### streakCount updated after check

Only **after** the check, it writes `userDataStorage.streakCount = 5`.
{% endstep %}

{% step %}

### Net result

Net: user is **denied** the jackpot **today**, despite meeting the requirement.
{% endstep %}
{% endstepper %}

## References

* `handleRandomness()` source: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L207-L265>
