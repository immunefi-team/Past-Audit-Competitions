# 50951 sc high inconsistent streak count usage between jackpot and raffle ticket calculations

**Submitted on Jul 29th 2025 at 23:32:59 UTC by @KlosMitSoss for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50951
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

Whenever the randomness callback from the Supra Router is handled, the current streak of the user is calculated. However, this newly calculated streak count is only used for the raffle ticket calculation, not for checking if it is large enough to claim the jackpot. As a result, either the user receives more raffle tickets than intended or the user is not able to claim the jackpot.

### Vulnerability Details

When the current day is the day after the last spin, the streak count is incremented when it is computed in `Spin::handleRandomness()`. As a result, the spin might be enough to claim the jackpot when it is won. However, the check is done using the old streak count as it is only updated afterward (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L253>).

However, when raffle tickets are won instead and the amount of raffle tickets is calculated, the incremented streak count is used. This is clearly inconsistent.

To mitigate this, either use the old streak count in `Spin::determineReward()` or use the incremented streak count when checking if the streak count is enough to claim the jackpot. Either way, the inconsistency should be removed.

### Impact Details

Claiming the jackpot will not be possible even though the current streak count is large enough. On the other hand, if it is intended to use the old streak count, users winning raffle tickets will receive more than intended.

This means either users lose out on a significant amount of funds due to the jackpot not being claimable when won or other users potentially losing out on raffle winnings due to other users receiving more tickets than intended and therefore lowering others' chances.

### References

Code references are provided throughout the report.

## Proof of Concept

{% stepper %}
{% step %}

### Start a spin

Alice calls `Spin::startSpin()` to initiate a spin (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L174-L195>).
{% endstep %}

{% step %}

### Randomness callback and streak computation

The Supra oracle calls `Spin::handleRandomness()` with the random number. The streak is computed (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L217>).
{% endstep %}

{% step %}

### Raffle ticket calculation uses incremented streak

The returned streak, which might be `userData[user].streakCount + 1` (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L322>) if the current day is the day after the last spin, is then used in `Spin::determineReward()` (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L219>). Within this function, the streak is used to calculate the number of raffle tickets Alice wins, in case the spin results in raffle tickets (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L298>).
{% endstep %}

{% step %}

### Jackpot check uses old streak

However, when Alice wins the jackpot, the streak count used to determine if it is sufficient to claim the jackpot is still the old streak count, without the increment (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L232>). This inconsistency might prevent the jackpot from being claimable.
{% endstep %}
{% endstepper %}
