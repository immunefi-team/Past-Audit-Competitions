# 51878 sc high timing misalignment between campaign days and calendar days allows double spinning on high probability jackpot days

**Submitted on Aug 6th 2025 at 12:09:12 UTC by @vivekd for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51878
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

### Brief/Intro

The Spin contract uses inconsistent day calculation methods for spin cooldowns versus jackpot probability determination, creating timing windows where a single high-probability campaign day spans multiple calendar days.

This allows users to bypass the intended daily spin limit by spinning on consecutive calendar days during the same profitable campaign day period, effectively doubling their chances at high-value jackpots and disrupting the lottery's economic balance.

## Vulnerability Details

The vulnerability stems from two different day calculation systems used within the same contract.

{% stepper %}
{% step %}

### Calendar Days (for spin restrictions)

The canSpin() modifier uses the DateTime library for actual calendar date boundaries:

```solidity
// Lines 149-162 in Spin.sol
  (uint16 lastSpinYear, uint8 lastSpinMonth, uint8 lastSpinDay) = (
      dateTime.getYear(_lastSpinTimestamp),
      dateTime.getMonth(_lastSpinTimestamp),
      dateTime.getDay(_lastSpinTimestamp)
  );

  (uint16 currentYear, uint8 currentMonth, uint8 currentDay) =
      (dateTime.getYear(block.timestamp), dateTime.getMonth(block.timestamp), dateTime.getDay(block.timestamp));

  if (isSameDay(lastSpinYear, lastSpinMonth, lastSpinDay, currentYear, currentMonth, currentDay)) {
      revert AlreadySpunToday();
  }
```

{% endstep %}

{% step %}

### Campaign Days (for jackpot probabilities)

The reward system uses timestamp arithmetic from campaign start:

```solidity
// Lines 285-290 in Spin.sol
  uint256 daysSinceStart = (block.timestamp - campaignStartDate) / 1 days;
  uint8 dayOfWeek = uint8(daysSinceStart % 7);
  uint256 jackpotThreshold = jackpotProbabilities[dayOfWeek];
```

{% endstep %}
{% endstepper %}

### The Critical Misalignment

When campaignStartDate is not aligned with midnight UTC boundaries, campaign day periods can span across multiple calendar days. This creates exploitation windows where users can spin multiple times during a single high-probability campaign day period.

Example scenario:

* Campaign starts Sunday 12:00 PM UTC.
* Campaign Day 6 (highest probability = 20) runs from Sunday 12:00 PM to Monday 12:00 PM — spanning the calendar day boundary at midnight.
* A user can spin Sunday 11:59 PM (calendar Sunday) and again Monday 12:01 AM (calendar Monday) while both spins fall into the same campaign day (Day 6) and thus enjoy identical high jackpot odds.

### Economic Impact

The jackpot probabilities array \[1, 2, 3, 5, 7, 10, 20] shows dramatic variation, with the highest probability day offering 20x better odds than the lowest. Users exploiting this timing can effectively double their spins on the most profitable days of the week, accelerating reward pool depletion and potentially causing operational issues.

{% hint style="danger" %}
Impact: Fast reward pool depletion and potential operational shutdown of the lottery system due to excessive payouts or drained funds.
{% endhint %}

## Proof of Concept

Prerequisites:

* Campaign starts at a non-midnight time creating calendar/campaign day misalignment
* Jackpot probabilities vary significantly across days of the week
* User has sufficient PLUME tokens for multiple spin fees

{% stepper %}
{% step %}

### Identify Misalignment Window

* Example: Campaign starts Sunday 12:00 PM UTC (campaignStartDate = 1704715200).
* Campaign day boundaries occur at noon; calendar days at midnight.
* This creates a 12-hour overlap window twice per week.
  {% endstep %}

{% step %}

### Target High-Probability Campaign Day

* Campaign Day 6 has jackpot probability of 20 (highest in the week).
* This day spans: Sunday 12:00 PM UTC to Monday 12:00 PM UTC.
* Window crosses calendar day boundary at Sunday midnight.
  {% endstep %}

{% step %}

### Execute First Spin

* Time: Sunday 11:59 PM UTC.
* Campaign day calculation: (Sunday\_11:59PM - Sunday\_12:00PM) / 86400 = 0.499 → Campaign Day 0, which maps to index 6 in weekly cycle.
* Calendar day: Sunday.
* Jackpot probability: jackpotProbabilities\[6] = 20.
* Spin executes successfully with 20x jackpot odds.
  {% endstep %}

{% step %}

### Execute Second Spin

* Time: Monday 12:01 AM UTC (2 minutes later).
* Campaign day calculation: (Monday\_12:01AM - Sunday\_12:00PM) / 86400 = 0.501 → Still Campaign Day 0, index 6.
* Calendar day: Monday.
* Same jackpot probability: jackpotProbabilities\[6] = 20.
  {% endstep %}

{% step %}

### Bypass Verification

* canSpin() check compares calendar days: Sunday ≠ Monday → passes.
* Second spin is allowed despite being in the same high-probability campaign period.
  {% endstep %}

{% step %}

### Exploitation Result

* User achieves 2 spins with 20x jackpot probability instead of intended 1 spin.
* Effective advantage: 2 × 20x = 40x expected value for that campaign day.
* Both spins occurred within a 2-minute window during peak probability period.
  {% endstep %}
  {% endstepper %}

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L137-L165>

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L267-L304>
