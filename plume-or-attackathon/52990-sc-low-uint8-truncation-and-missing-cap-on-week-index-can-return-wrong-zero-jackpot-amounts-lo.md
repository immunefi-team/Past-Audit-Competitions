# 52990 sc low uint8 truncation and missing cap on week index can return wrong zero jackpot amounts low contract fails to deliver promised returns&#x20;

* **Submitted on:** Aug 14th 2025 at 15:18:09 UTC by @daxun for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #52990
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>

## Description

### Brief / Intro

`determineReward()` computes:

```solidity
uint8 weekNumber = uint8(getCurrentWeek());
```

* Past the designed 12-week campaign, `jackpotPrizes[weekNumber]` often defaults to **0** (uninitialized), so winners can receive **0** unexpectedly.
* After **256 weeks**, casting to `uint8` **wraps** (mod 256) and can index an unintended bucket, again returning **wrong prizes** (commonly **0**).

### Vulnerability Details

* `initialize()` only configures indices `0..11`. No guard exists in `determineReward()` to zero out jackpots post-campaign or cap the index.
* `getWeeklyJackpot()` caps the display (`if (weekNumber > 11) return (weekNumber, 0, 0);`), but **this guard is not used** in `determineReward()`.

### Impact Details

{% hint style="info" %}

* Immunefi Impact: **Low — Contract fails to deliver promised returns, but doesn't lose value**
* After week 12 (or any misconfiguration where `campaignStartDate` is old), jackpot payouts can become **0**. After week 256, indexing is effectively **nondeterministic modulo 256**.
  {% endhint %}

## References

* `determineReward()` (`uint8 weekNumber = uint8(getCurrentWeek());`)
* `initialize()` (only sets 0..11)
* `getWeeklyJackpot()` (has a cap that isn’t applied in `determineReward()`)

`determineReward()` (<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L286>)

## Proof of Concept

{% stepper %}
{% step %}

### Setup: campaign start > 12 weeks ago

* Set `campaignStartDate` to be more than 12 weeks in the past.
  {% endstep %}

{% step %}

### Force a jackpot and trigger a spin

* Make `jackpotProbabilities[dayOfWeek]` high (or otherwise force a jackpot) and trigger a spin.
  {% endstep %}

{% step %}

### observe jackpot index and payout

* `determineReward()` computes `weekNumber` via `uint8(getCurrentWeek())`.
* Because `weekNumber > 11`, `jackpotPrizes[weekNumber]` is uninitialized (defaults to `0`).
* The user “wins” a **0-amount** jackpot.
  {% endstep %}

{% step %}

### Long-term wraparound (> 256 weeks)

* If `campaignStartDate` is more than 256 weeks ago, `uint8(getCurrentWeek())` wraps (mod 256) and indexes an unintended bucket (often unset = 0), producing wrong payouts.
  {% endstep %}
  {% endstepper %}
